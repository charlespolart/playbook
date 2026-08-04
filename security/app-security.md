# Sécurité applicative — checklist code (backend, front, mobile)

Le modèle mental : **tout ce qui vient du client est hostile** — body, query,
params, headers, cookies, fichiers, webhooks. La validation côté UI n'existe
pas pour un attaquant (curl parle directement à l'API). La sécurité se joue en
couches : même si une barrière saute, la suivante limite les dégâts.

## OWASP Top 10:2025 → notre stack

| OWASP 2025 | Chez nous, concrètement |
|---|---|
| A01 Broken Access Control | IDOR : chaque query filtrée par `user_id` (section AuthZ) |
| A02 Security Misconfiguration | ports Docker, headers, CORS (cette fiche + `vps-hardening.md`) |
| A03 Software Supply Chain | lockfile + audit deps (section Dépendances) |
| A04 Cryptographic Failures | hash/chiffrement : `data-protection.md` |
| A05 Injection | Drizzle paramétrise, sauf `sql.raw` (section Injection) |
| A06 Insecure Design | rate limit, enumeration, logique métier abusable |
| A07 Authentication Failures | argon2id, sessions, anti-bruteforce (section AuthN) |
| A08 Data Exposure | pas de PII/secrets dans logs et réponses d'erreur |
| A09 Logging & Alerting | logger les échecs d'auth, savoir qu'on se fait attaquer |
| A10 Mishandling of Exceptional Conditions | erreurs génériques au client, fail closed |

## Injection (SQL, shell)

- Drizzle paramétrise **par défaut** : le query builder ET le template
  ``sql`... ${valeur}` `` (les interpolations deviennent des `$1`). Les deux
  trous : **`sql.raw()`** avec de l'input user, et la **concaténation de
  string** avant de passer à `sql`.
- Cas piège classique : `ORDER BY` / nom de colonne dynamique — un identifiant
  ne PEUT PAS être paramétré. Allowlist obligatoire :
  ```ts
  const sortable = { name: users.name, createdAt: users.createdAt } as const;
  const col = sortable[input.sort]; // undefined si hors liste → 400
  ```
- Shell : jamais de commande construite avec de l'input user. `Bun.spawn(["cmd",
  arg])` / `execFile` (arguments en tableau), jamais `exec("cmd " + input)`.

## Validation aux frontières + mass assignment

- Un schéma zod par endpoint, `.strict()` pour rejeter les clés inconnues.
- **Jamais** `db.insert(users).values(body)` ni `db.update(...).set(body)` avec
  le body brut : un client ajoute `"role": "admin"` ou `"credits": 9999` et ça
  passe. Toujours passer par l'objet **explicite** sorti du schéma zod.
- **Le point aveugle : les endpoints de sync / bulk / import.** Un endpoint
  REST écrit 5 champs nommés, on le relit vite. Un endpoint de synchro reçoit
  « une ressource » générique — le schéma finit en
  `payload: z.record(z.string(), z.unknown())` et le handler recopie **toutes**
  les clés dans l'insert ORM. Forcer `id`/`ownerId`/`updatedAt` par-dessus ne
  suffit pas : toute **autre** colonne serveur reste écrivable (vécu :
  `isShared`, qui ne devait basculer que par le flux de partage), et les bornes
  `.max()/.trim()` des chemins REST sont contournées. Pas d'IDOR si l'ownership
  est vérifié — c'est une faille d'**intégrité**, plus discrète et donc plus
  durable. Parade : une **allowlist de colonnes dérivée du schéma**, pour
  qu'elle ne dérive jamais :
  ```ts
  const SERVER_OWNED = new Set(["id", "ownerId", "isShared", "createdAt",
                                "updatedAt", "deletedAt", "originDevice"]);
  const writable = new Set(
    Object.keys(getTableColumns(table)).filter((c) => !SERVER_OWNED.has(c)),
  );
  // puis: ignorer toute clé du payload absente de `writable`
  ```
- Webhooks entrants = input hostile aussi : vérifier l'authentification du
  webhook AVANT de parser (cf. `third-party/revenuecat.md` — header
  `Authorization` statique, pas de HMAC).

## AuthN

- Hash de mot de passe : `Bun.password.hash()` / `Bun.password.verify()` —
  argon2id par défaut, zéro dépendance. Jamais de md5/sha256 direct, jamais de
  crypto maison.
- Sessions : cookie `HttpOnly; Secure; SameSite=Lax`, id aléatoire ≥ 128 bits,
  invalidation **côté serveur** au logout (supprimer la row session, pas juste
  le cookie).
- Si JWT : algorithme **fixé** à la vérification (jamais lu depuis le header du
  token), secret ≥ 256 bits, `exp` courte + refresh token révocable.
  La forme qu'on trouve en vrai n'est pas `alg: none` — c'est
  ``algorithms: [decodeProtectedHeader(token).alg]`` (vécu 2026-08) : le code
  *croit* épingler alors qu'il **renvoie au token le droit de choisir**. Sur une
  vérif OIDC via JWKS distant c'est peu exploitable (le JWKS ne sert que des
  clés RSA/EC, une confusion HMAC meurt sur un mismatch de type de clé), mais
  la correction est gratuite : `algorithms: ["RS256", "ES256"]` en dur, plus un
  rejet préalable si l'`alg` annoncé n'est pas dans la liste. Le payload
  est **lisible par tous** (base64, pas chiffré) — jamais de donnée sensible
  dedans. Piège structurel : un JWT pur ne se révoque pas avant son `exp` → si
  besoin d'un "logout partout", il faut de l'état serveur de toute façon.
- Messages d'erreur neutres : « identifiants invalides », jamais « email
  inconnu » (énumération des comptes — vaut aussi pour /forgot-password : même
  réponse que l'email existe ou non).

## AuthZ — IDOR (le n°1 OWASP depuis 2021)

- Toute ressource chargée par id doit être filtrée par **ownership** :
  ```ts
  where(and(eq(docs.id, input.id), eq(docs.userId, session.userId)))
  ```
  Jamais `eq(docs.id, input.id)` seul en se disant que l'id est introuvable —
  les ids s'énumèrent (même les UUID fuient dans les URLs partagées).
- Le rôle/plan vient de la **DB via la session**, jamais du client.
- Endpoints admin : check serveur sur **chaque** endpoint. Cacher le bouton
  dans l'UI n'est pas un contrôle d'accès.

## XSS

- React échappe par défaut → le sujet ne se pose que sur
  `dangerouslySetInnerHTML` (et le rendu de markdown user). Si inévitable :
  DOMPurify, pas de regex maison.
- Deuxième couche : CSP via Caddy (snippet headers ci-dessous). Déployer
  d'abord en `Content-Security-Policy-Report-Only` pour voir ce qui casse
  (les SPA Vite ont souvent besoin de `style-src 'unsafe-inline'`).

## CSRF & SSRF

- CSRF : `SameSite=Lax` couvre le cas API même-site + SPA. Si un jour cookies
  cross-site (`SameSite=None`) → tokens CSRF obligatoires.
- SSRF : tout fetch serveur d'une URL fournie par l'user (import d'avatar par
  URL, webhook sortant configurable…) → allowlist de domaines et refus des IP
  privées/link-local (`169.254.169.254` = metadata cloud). S'il n'y a pas de
  feature de ce genre, ne pas en créer une par accident.

## Uploads

- Limite de taille, allowlist d'extensions ET vérif des magic bytes (le
  Content-Type client est déclaratif), nom de fichier régénéré aléatoire
  (jamais le nom client → path traversal `../`).
- Servir depuis un bucket/domaine séparé (R2), jamais depuis le webroot du
  backend, jamais exécutable.

## Headers de sécurité (Caddy)

```caddyfile
{$DOMAIN} {
    header {
        Strict-Transport-Security "max-age=31536000"
        X-Content-Type-Options nosniff
        Referrer-Policy strict-origin-when-cross-origin
        Permissions-Policy "camera=(), microphone=(), geolocation=()"
        Content-Security-Policy "default-src 'self'; img-src 'self' data: https:; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'"
        -Server
    }
    # ... handle @backend / handle SPA (cf. github-actions-deploy.md)
}
```

- `frame-ancestors 'none'` remplace `X-Frame-Options` (anti-clickjacking).
- Vérifier le résultat réel : https://securityheaders.com après deploy.

## CORS

- Liste **explicite** d'origins. Jamais `*` avec `credentials: true` (les
  navigateurs le refusent, et les "solutions" type refléter l'Origin reçu
  reviennent à désactiver CORS).
- Une app mobile ne passe pas par CORS — inutile d'élargir pour Flutter.

## Rate limiting

- Middleware applicatif sur les routes sensibles : /login, /signup,
  /forgot-password, et tout endpoint coûteux. Par IP **et** par compte (un
  botnet a beaucoup d'IPs, un compte ciblé n'en a qu'un).
- Piège derrière Caddy : le backend voit l'IP du réseau Docker (172.x), pas le
  client. Lire `X-Forwarded-For` — acceptable **uniquement** parce que Caddy
  est le seul point d'entrée (le header est réécrit par lui) ; le jour où le
  backend est exposé direct, ce header devient spoofable.
- **Piège d'ordre des middlewares (vécu, Listable 2026-07)** : un keyer « par
  userId sinon IP » ne marche que si la session est **chargée avant** le
  bucket. Un rate-limit enregistré app-level tourne avant le `requireAuth()`
  des sous-routers → `c.get("userId")` est toujours vide et tout le monde
  partage le budget IP (NAT = faux positifs, et la limite par compte n'existe
  pas). Fix : un `optionalAuth()` app-level AVANT le bucket, et `requireAuth`
  réutilise le contexte déjà chargé (un seul `getSession` par requête). Le bug
  est **silencieux** : les 429 partent quand même, juste sur la mauvaise clé —
  tester avec deux comptes derrière la même IP.

## Secrets côté code

- `.env` gitignored partout ; un secret commité = fuité → **rotation**, pas
  juste `git rm` (l'historique le garde). Scanner : gitleaks en CI.
- Flutter / dart-define : **tout ce qui entre dans le binaire mobile est
  extractible** (strings, décompilation). N'y mettre que des clés conçues pour
  être publiques (clé publique RevenueCat, config Firebase). Les secrets
  restent côté serveur — l'app mobile ne doit jamais porter un secret que le
  backend pourrait porter à sa place.
- Tokens côté mobile : Keychain/Keystore (`flutter_secure_storage`), pas
  SharedPreferences.
- **Flutter : `debugPrint` s'exécute AUSSI en release** (c'est un `print`
  throttlé, pas un no-op — contrairement à ce que le nom laisse croire). Tout
  `debugPrint` qui dumpe un body de réponse ou un payload part donc dans
  logcat / os_log en production. Garder les logs verbeux derrière
  `if (kDebugMode)`, et grep les `debugPrint` avant release.
- **Permissions Android/iOS : ne déclarer que ce qui est réellement demandé par
  le code.** Vérifier dans le **plugin natif**, pas dans son README : un
  `image_picker` moderne ne réclame que `CAMERA` (la galerie passe par un
  picker système qui accorde l'accès par URI), donc `READ_MEDIA_IMAGES` est
  superflue — et Play Console la fera justifier. Idem côté iOS, une clé
  `NSFaceIDUsageDescription` sans aucune biométrie dans l'app, ou une string
  caméra qui parle de scan de codes-barres inexistant, sont des frictions
  gratuites en App Review.
- **`android:allowBackup`** vaut `true` par défaut : les données de l'app
  partent dans Auto Backup et sont extractibles par `adb backup`. Le mettre à
  `false` (ou fournir des `dataExtractionRules`) dès que l'app stocke autre
  chose que du cache.

## Dépendances (A03:2025 — supply chain)

- Lockfile committé + `--frozen-lockfile` en CI. **Ne pas croire que c'est
  « déjà le cas »** : audité 2026-07-12, la CI de Listable faisait un
  `bun install` nu depuis le début — vérifier le workflow, pas la doctrine.
- `bun audit` (ou Dependabot alerts sur le repo GitHub — gratuit, à activer).
- Avant d'ajouter une dep : dernier commit, downloads, nombre de deps
  transitives. Une dep = du code tiers qui tourne avec les droits du backend.

## Logs & erreurs

- En prod : erreur générique au client (`500` + id de corrélation), stack
  trace dans les logs serveur uniquement.
- Jamais de password/token/PII dans les logs (les logs partent en clair dans
  journald/stdout et vivent longtemps).
- Logger les **échecs** d'auth et les 403 — c'est ce qui permet de voir un
  bruteforce pendant qu'il a lieu et pas après.

### Un secret dans l'URL échappe à la redaction (vécu, 2026-07)

La `redact` d'un logger (pino & co) masque des **clés d'objet**
(`req.headers.authorization`, `*.token`). Elle ne verra **jamais** un secret
encapsulé dans une *valeur string* — typiquement un token porteur d'accès qui
voyage dans le **chemin** : `/i/:token`, `/invitations/:token/preview`,
`/reset/:token`, `/unsubscribe/:token`. Le logger de requêtes écrit
`path: c.req.path` et le secret part en clair dans journald… et dans Sentry.

Deux règles :

1. **Masquer le segment à la source**, avant qu'il n'atteigne le log :
   ```ts
   export function redactPath(path: string): string {
     return path
       .replace(/^(\/i)\/[^/]+/, "$1/[redacted]")
       .replace(/(\/invitations\/by-token)\/[^/]+/, "$1/[redacted]");
   }
   ```
2. **Recenser TOUS les sinks**, c'est là que ça se joue. Sur un cas réel il y
   en avait **cinq**, et une première passe n'en avait couvert que trois : le
   logger de requêtes, le handler `notFound`, le `onError`… mais aussi le
   **middleware de timeout** et le **middleware Sentry** — ce dernier montant
   avant `onError` et expédiant donc le token à un **service externe**. Les
   deux manquants ont été trouvés par une relecture dédiée, pas par le premier
   passage.
   ```bash
   # doit ne rien renvoyer
   grep -rn "req\.path" src/ | grep -v redactPath
   ```

Corollaire de conception : un secret porteur d'accès a moins de surface dans un
**header** ou un **body POST** que dans un chemin ou une query string (chemins
et query strings finissent dans les access logs, les referers, l'historique des
proxys). Si le design le permet, éviter le token dans l'URL — un audit voisin a
trouvé le même problème sur une **WebSocket** (`ws://…?token=…`) et a dû
migrer vers un premier message `{"type":"auth","token":…}`, avec période de
double compatibilité car les apps mobiles déjà installées continuent d'envoyer
l'ancien format pendant des mois.

## Checklist avant mise en prod

- [ ] Chaque endpoint : zod `.strict()` + filtre ownership
- [ ] Endpoints sync/bulk/import : allowlist de colonnes, pas de `z.unknown()` recopié
- [ ] `Bun.password` pour les hash, sessions HttpOnly/Secure/SameSite
- [ ] Rate limit sur login/signup/forgot
- [ ] Headers Caddy posés, vérifiés sur securityheaders.com
- [ ] CORS = liste explicite
- [ ] Aucun secret dans le repo ni dans le binaire mobile (gitleaks passe)
- [ ] Erreurs prod génériques, logs sans PII
- [ ] Aucun token dans un chemin loggé : `grep -rn "req\.path" src/ | grep -v redact` vide
- [ ] `security/vps-hardening.md` déroulée côté infra

## Liens

- OWASP Top 10:2025 : https://owasp.org/Top10/2025/
- OWASP Cheat Sheet Series (LA référence par sujet, dense et à jour) :
  https://cheatsheetseries.owasp.org/
- Test headers : https://securityheaders.com

---

**Dernière vérif** : 2026-08-04 (audits croisés de 2 backends en prod :
secrets d'invitation en clair dans 5 sinks de logs dont Sentry, mass assignment
sur un endpoint de sync, `algorithms: [alg]` lu depuis le token, `debugPrint`
de PII en release, IDOR réel sur un endpoint de reorder côté projet voisin).
