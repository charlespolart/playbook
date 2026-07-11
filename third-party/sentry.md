# Sentry — crash reporting & erreurs (backend + Flutter)

## Décision (actée 2026-07-12, projet Listable)

**Sentry, pas Crashlytics**, dès qu'un projet a un backend self-hosté :

- **Full-stack** : Crashlytics est mobile-only ; il ne verra jamais le Bun/Node
  du VPS. Le choisir = deux outils, deux dashboards, zéro corrélation
  front/back. Sentry couvre les deux dans la même orga.
- **Indépendance** : open-source, self-hostable sur le VPS si le pricing/quota
  agace un jour. Crashlytics = Google propriétaire à vie.
- **Résidence EU** disponible au signup (RGPD plus propre que les SCC Google).
- **Erreurs Dart** : grouping, breadcrumbs et non-fatals nettement meilleurs
  que Crashlytics (qui excelle surtout sur les crashs *natifs* et le volume
  gratuit illimité — pertinent seulement pour du mobile pur à grosse échelle).

Limite à connaître : **free tier 5 000 événements/mois**. Une crash-loop chez
une poignée d'users le mange en un jour → configurer sampling + rate limit SDK
dès la pose, pas après le premier dépassement.

## Backend (pattern validé en prod sur Listable)

- Module `observability/sentry.ts` : no-op si `SENTRY_DSN` absent (dev + CI
  tournent sans compte). `APP_RELEASE` = SHA git stampé au build → chaque
  erreur est rattachée à sa release.
- Capture dans le `onError` global avec `requestId` en contexte — c'est ce qui
  permet de retrouver la requête exacte dans les logs pino.

## Flutter (à la première pose — PAS ENCORE VÉCU, compléter alors)

- `sentry_flutter` ; init en wrappant `runApp` (`SentryFlutter.init(..., appRunner:)`).
  DSN vide en debug → SDK inerte, même convention que le backend.
- **Upload des debug symbols** (dSYM iOS / mapping Android) via `sentry-cli`
  dans le lane Fastlane de release, sinon stack traces illisibles.
- `beforeSend` pour scrubber les PII avant envoi (emails, noms de listes…).
- Capturer les **échecs avalés** en priorité : tout `catch` silencieux d'un
  flux critique (application de patch WS, sync) doit faire un
  `Sentry.captureException` — c'est là que vivent les bugs invisibles.
- RGPD : Sentry devient un **sous-traitant** → l'ajouter à la liste des
  destinataires de la privacy policy (cf. `legal/rgpd.md`).

## Symptômes à reconnaître

- « L'app crashe chez un utilisateur et on ne l'apprend jamais » → pas de
  crash reporting mobile ; poser cette fiche.
- Stack traces Sentry mobile illisibles (adresses hexa) → symbols pas uploadés
  dans le build release.
- Quota Sentry explosé en un jour → crash-loop + pas de rate limit SDK.

Doc officielle : <https://docs.sentry.io/platforms/flutter/> ·
<https://docs.sentry.io/platforms/javascript/guides/bun/>

*Dernière vérif : 2026-07-12 (décision + pattern backend vécus ; section
Flutter théorique, à confirmer à la première intégration).*
