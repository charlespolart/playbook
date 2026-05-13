# RevenueCat — webhook backend

## Le piège

**RevenueCat ne signe pas le body des webhooks en HMAC.** Le header `x-revenuecat-signature` cité dans certains wrappers tiers / vieille doc / threads de forum **n'existe pas** sur les webhooks RC officiels.

Si tu codes le handler en partant des conventions de l'industrie (Stripe, GitHub, Shopify — qui utilisent tous HMAC-SHA256), ton handler va `401` tous les vrais webhooks et zéro souscription ne se synce. Le pire : ça marche en test si tu fakes toi-même la signature, ça casse silencieusement en prod.

C'est exactement ce qui était shippé dans Listable jusqu'au commit `53554e2`.

## Le bon pattern

RC supporte **un seul mécanisme d'auth** : une valeur statique configurée dans le dashboard, envoyée verbatim dans le header `Authorization`. C'est documenté ici :
- https://www.revenuecat.com/docs/integrations/webhooks (section "Security")
- Confirmé en 2026 sur le forum RC : https://community.revenuecat.com/dashboard-tools-52/is-x-revenuecat-signature-removed-and-where-is-webhook-secret-key-7110

Côté handler backend, ça donne :

```ts
import { timingSafeEqual } from "node:crypto";

function verifyAuthorization(header: string | undefined): boolean {
  if (!header) return false;
  const secret = env.REVENUECAT_WEBHOOK_SECRET;
  if (!secret) return false; // fail closed
  const a = Buffer.from(header, "utf8");
  const b = Buffer.from(secret, "utf8");
  if (a.length !== b.length) return false; // timingSafeEqual exige same length
  return timingSafeEqual(a, b);
}

app.post("/webhooks/revenuecat", async (c) => {
  if (!verifyAuthorization(c.req.header("authorization"))) {
    return c.json({ error: "unauthorized" }, 401);
  }
  // ... handle event
});
```

Points-clés :
- `timingSafeEqual` (constant-time) pour éviter les attaques par timing
- Short-circuit sur length mismatch avant `timingSafeEqual` (qui throw sinon)
- Fail closed quand `REVENUECAT_WEBHOOK_SECRET` est vide (évite que la prod boote en accept-all)
- On lit le header `authorization` minuscule (Hono normalise les headers en lowercase)

## Setup côté dashboard RC

1. Génère le secret côté backend : `openssl rand -base64 48`
2. Mets-le dans `.env` prod sous `REVENUECAT_WEBHOOK_SECRET=…`
3. Redéploie pour que le backend pick up
4. Dashboard RC → projet → **Integrations** → **+ New** → **Webhook**
   - **Webhook name** : `<projet>-prod`
   - **Webhook URL** : `https://<your-api-host>/api/v1/webhooks/revenuecat`
   - **Authorization header value** : colle le **secret nu** (PAS le format `Bearer <secret>` suggéré dans le placeholder). RC envoie verbatim ce que tu tapes dans le header `Authorization`, et le backend fait un compare brut. Donc des deux côtés : exactement la même string, sans préfixe. Évite la confusion.
   - **Environment to send events for** : **Production** uniquement, jamais "Both Production and Sandbox" pour un webhook prod. Sinon les achats sandbox (TestFlight, Play test track) vont être traités comme de vrais achats → DB prod polluée + faux comptes premium + analytics bidons.
   - **App / Event type** : laisse "All apps" + "All events". Le backend filtre déjà les types qu'il connaît et no-op les inconnus.
5. Bouton **"Send test event"** dans la config du webhook → un 200 doit arriver et tu dois voir un log `revenuecat: processed` ou `ignored` côté backend.

## Idempotence (à pas oublier)

RC re-livre un event si tu ne réponds pas en `2xx` sous 60s. Donc ton handler doit être **idempotent** :
- Dédupliquer par `transaction_id` (ou `event.id`) avec `ON CONFLICT DO NOTHING` sur ta table `purchase_history`
- Calculer l'état suivant à partir du type d'event, pas par incrément (`UPSERT` plutôt que `UPDATE … SET balance = balance + …`)

## Anonymous IDs

RC envoie parfois un `app_user_id` qui commence par `$RCAnonymousID:` — c'est un user pas encore identifié côté SDK mobile. À no-op (retourner `200 { ignored: true }`) jusqu'à ce que `Purchases.logIn(userId)` ait été call côté app et qu'un vrai event arrive avec le `user_id` backend.

## Tester localement

Pour test runs : mock un POST avec le bon `Authorization` header et le body JSON RC. Pas besoin de simuler la signature. Exemple Bun test :

```ts
function authedBody(body: unknown) {
  return {
    body: JSON.stringify(body),
    authorization: process.env.REVENUECAT_WEBHOOK_SECRET ?? "",
  };
}
```

## Source de vérité côté frontend

Le webhook met à jour la table backend. Le frontend cache localement l'état premium (via SharedPreferences / equivalent) mis à jour par le listener RC SDK. Pour cross-check : endpoint `GET /me/subscription` qui lit la DB, appelé en backup au boot de l'app si le listener a raté un event.

---

**Dernière vérif** : 2026-05-13 (Listable + dian-dian, vs. doc RC officielle).
**Projets concernés** : `listable`, `dian-dian` (pattern identique).
