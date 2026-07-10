# Sign in — Better Auth + Apple/Google (natif, `id_token`-only)

Le pattern d'auth validé sur **Listable** (backend Hono + Better Auth, mobile Flutter),
repris sur **unfold**. À lire avant de câbler un « Se connecter avec Apple/Google ».

## Le pattern qui marche : `id_token`-only, pas de redirection web

Le mobile obtient l'`id_token` **nativement** (packages `sign_in_with_apple` /
`google_sign_in`), l'envoie au backend, et **Better Auth vérifie ce JWT contre les
JWKS publics du provider**. Pas de flux OAuth web (pas de WebView, pas de redirect,
pas de deep-link à gérer). Le mobile n'appelle jamais Apple/Google en OAuth — juste
le SDK natif pour récupérer l'`id_token`, puis `POST …/sign-in/social {provider, idToken}`.

## LE piège Apple : **tu n'as PAS besoin de clé `.p8`**

En flux natif `id_token`, la connexion Apple ne demande **ni Services ID, ni clé
`.p8`, ni client secret, ni rotation 6 mois**. Il te faut seulement :
- le **bundle id** (ex. `com.overridedev.unfold`) → c'est l'`aud` de l'`id_token` natif ;
- le **Team ID** (Apple) ;
- l'**App ID** + la capability *Sign in with Apple* → **gérés dans Xcode** (auto-signing), Charles n'a rien à créer sur le portail.

Le client secret (JWT signé par la `.p8`) ne sert qu'à **appeler `/auth/token`**
(code exchange, non utilisé) **ou `/auth/revoke`** (révocation). Voir la nuance ci-dessous.

⚠️ **Attention : deux clés `.p8` distinctes** sur le portail Apple — la clé **APNs**
(notifications push) ≠ la clé **Sign in with Apple** (auth). Ne pas les confondre.

### La révocation à la suppression de compte : théorie vs pratique
- **Officiellement**, Apple exige : si l'app propose Sign in with Apple **et** la
  suppression de compte (obligatoire, guideline **5.1.1(v)**), il faut **révoquer les
  tokens via l'API REST `/auth/revoke`** — ce qui redemande le client secret (donc la `.p8`).
  Sources : [TN3194](https://developer.apple.com/documentation/technotes/tn3194-handling-account-deletions-and-revoking-tokens-for-sign-in-with-apple) ·
  [Token revocation](https://developer.apple.com/documentation/sign_in_with_apple/revoke_tokens/) ·
  [Account deletion requirement](https://developer.apple.com/support/offering-account-deletion-in-your-app/).
- **En pratique**, **Listable** a la suppression de compte (`DELETE /api/v1/account`) **et**
  Sign in with Apple, **sans clé ni révocation** — et passe la review. Ce call est
  serveur-à-serveur, **invisible depuis l'app**, donc non testé par le reviewer.
- **Décision** : on part **sans clé** (comme Listable). Si Apple durcit un jour, ajouter
  la révocation est un petit ajout backend (générer le client secret JWT + appeler
  `/auth/revoke` dans le service de suppression). Ne pas bloquer le dev là-dessus.

## Config backend (Better Auth)

Le point non-évident : **le vérificateur d'`id_token` intégré de Better Auth n'expose
aucun réglage de tolérance d'horloge**. Une désync d'**1 seconde** entre ta box et l'IdP
→ `jwtVerify` throw « iat must be in the past » → **401 INVALID_TOKEN**. On **override
`verifyIdToken` par provider** avec ~5 s de tolérance (escape hatch documenté).

```ts
import { createRemoteJWKSet, decodeProtectedHeader, jwtVerify } from "jose";

const ID_TOKEN_CLOCK_TOLERANCE = "5s";
const GOOGLE_JWKS = createRemoteJWKSet(new URL("https://www.googleapis.com/oauth2/v3/certs"));
const APPLE_JWKS  = createRemoteJWKSet(new URL("https://appleid.apple.com/auth/keys"));

async function verifyOidcToken(token, jwks, issuers, audience) {
  try {
    const { alg } = decodeProtectedHeader(token);
    if (!alg) return false;
    await jwtVerify(token, jwks, {
      algorithms: [alg], issuer: issuers, audience,
      maxTokenAge: "1h", clockTolerance: ID_TOKEN_CLOCK_TOLERANCE,
    });
    return true;
  } catch { return false; }
}

// Google — audience = le client id
const google = env.GOOGLE_CLIENT_ID && env.GOOGLE_CLIENT_SECRET ? {
  clientId: env.GOOGLE_CLIENT_ID, clientSecret: env.GOOGLE_CLIENT_SECRET,
  verifyIdToken: (t) => verifyOidcToken(t, GOOGLE_JWKS,
    ["https://accounts.google.com", "accounts.google.com"], env.GOOGLE_CLIENT_ID),
} : undefined;

// Apple — natif : aud == bundle id ; web : aud == Services ID. On accepte les deux.
const APPLE_BUNDLE_ID = "com.overridedev.<app>";
const apple = env.APPLE_CLIENT_ID ? {
  clientId: env.APPLE_CLIENT_ID,
  appBundleIdentifier: APPLE_BUNDLE_ID, // sinon tout sign-in natif échoue en INVALID_TOKEN
  verifyIdToken: (t) => verifyOidcToken(t, APPLE_JWKS,
    "https://appleid.apple.com", [APPLE_BUNDLE_ID, env.APPLE_CLIENT_ID]),
} : undefined;

export const auth = betterAuth({
  // …drizzleAdapter, emailAndPassword, session cookie HTTP-only 30j…
  socialProviders: { ...(google && { google }), ...(apple && { apple }) },
});
```

Providers montés **conditionnellement** (si les env credentials sont présents) → pas de
crash au boot sans credentials, et on active les sociaux quand ils arrivent.

### Autres gotchas Better Auth (validés sur Listable)
- `advanced.database.generateId: false` — laisse Postgres générer l'id (`defaultRandom()` uuid).
  Sinon Better Auth met sa string ~32 chars → `invalid input syntax for type uuid` → **422 à chaque sign-up**.
- `advanced.cookiePrefix: "<marque>"` — cookies `marque.session_token` au lieu de `better-auth.*`.
- `advanced.useSecureCookies: false` (même en prod) — quand `true`, Better Auth **préfixe
  `__Secure-`** au nom du cookie ; si l'intercepteur cookie mobile matche le préfixe nu
  (`marque.session_token=`), il **skippe silencieusement** tous les Set-Cookie signés →
  bug « TestFlight cold-boot perd la session ». Le flag Secure lui-même reste via
  `defaultCookieAttributes.secure`.
- `advanced.disableCSRFCheck: true` — le CSRF de Better Auth exige un header `Origin`/`Referer`
  dès qu'il y a un cookie ; Dart/Dio n'envoie pas d'`Origin`. Consommateurs = app mobile + curl
  (HTTPS + cookie en Keychain/Keystore) → le gate CSRF n'apporte rien. **Ré-activer si on
  sert un jour un flux web depuis le même backend.**

## Côté mobile (Flutter)
- Deps : `sign_in_with_apple`, `google_sign_in`. (Sur unfold, à **ajouter** au pubspec.)
- **Google Android** a besoin du **Web client id** comme `serverClientId` (pas le client id Android).
  Le passer via `--dart-define` (ex. `GOOGLE_SERVER_CLIENT_ID`). iOS le lit du plist / `GoogleService-Info`.
- Flow : SDK natif → `id_token` → `POST ${api}/…/sign-in/social` avec `{ provider, idToken: { token } }`
  (cookie de session anon présent si on lie un compte anonyme). Le cookie de session renvoyé est
  persisté (`flutter_secure_storage`) et réémis en header `cookie:` sur les appels suivants.

## Credentials à demander à Charles (checklist)
- **Bundle id définitif** (voir aussi [`mobile/apple-team-id-and-bundle-id.md`](../mobile/apple-team-id-and-bundle-id.md)) — sert d'`aud` Apple **et** de package Google iOS/Android.
- **Google** (console.cloud.google.com) : 3 OAuth client IDs (**iOS / Android / Web**) + le **Client Secret du Web**. Le Web (`GOOGLE_CLIENT_ID`/`SECRET`) → backend ; iOS/Android → configs natives ; le Web sert aussi de `serverClientId` Android.
- **Apple** : **Team ID** + le bundle id. **Pas de Services ID ni de `.p8`** en flux natif (voir plus haut). App ID + capability = Xcode.
- Les secrets (`GOOGLE_CLIENT_SECRET`, et la `.p8` si un jour révocation) → `/srv/<app>/.env`, **jamais** dans le repo.

---

*Dernière vérif : 2026-07-10 — Better Auth 1.6.x, packages `sign_in_with_apple` / `google_sign_in`. Source de vérité : le backend Listable (`apps/backend/src/auth/index.ts`). Exigence révocation Apple confirmée sur [developer.apple.com](https://developer.apple.com/support/offering-account-deletion-in-your-app/) mais non contrôlée en review (Listable passe sans).*
