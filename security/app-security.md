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
  token), secret ≥ 256 bits, `exp` courte + refresh token révocable. Le payload
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

## Dépendances (A03:2025 — supply chain)

- Lockfile committé + `--frozen-lockfile` en CI (déjà le cas).
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

## Checklist avant mise en prod

- [ ] Chaque endpoint : zod `.strict()` + filtre ownership
- [ ] `Bun.password` pour les hash, sessions HttpOnly/Secure/SameSite
- [ ] Rate limit sur login/signup/forgot
- [ ] Headers Caddy posés, vérifiés sur securityheaders.com
- [ ] CORS = liste explicite
- [ ] Aucun secret dans le repo ni dans le binaire mobile (gitleaks passe)
- [ ] Erreurs prod génériques, logs sans PII
- [ ] `security/vps-hardening.md` déroulée côté infra

## Liens

- OWASP Top 10:2025 : https://owasp.org/Top10/2025/
- OWASP Cheat Sheet Series (LA référence par sujet, dense et à jour) :
  https://cheatsheetseries.owasp.org/
- Test headers : https://securityheaders.com

---

**Dernière vérif** : 2026-07-08 (rédaction initiale ; Top 10:2025 final
vérifié en ligne ce jour).
