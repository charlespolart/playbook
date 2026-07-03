# Coding Playbook

Knowledge base partagée entre **Charles** (l'humain) et **les instances de Claude Code** qui bossent sur ses projets.

L'idée : éviter de re-tomber dans les mêmes pièges. Chaque fois qu'on découvre un truc non-évident — bug subtil dans une intégration tierce, gotcha d'un outil, convention qui marche bien sur un projet — on l'écrit ici. La prochaine fois (ou sur la prochaine app), on lit avant de coder.

## Pour les instances Claude Code

Quand on te demande d'intégrer un service tiers (RevenueCat, R2, Resend, Firebase, etc.), un pattern d'infra (Docker Compose, GHCR, deploy), ou une convention propre au workflow de Charles : **vérifie d'abord si une fiche existe ici**. Si oui, applique ce qui est dedans plutôt que de partir des suppositions par défaut. Si la fiche est incomplète ou contredit ton intuition basée sur la doc à jour, signale-le avant de coder.

Quand tu apprends quelque chose qui mérite d'être capturé (bug récurrent, contre-intuitif, ou simplement une convention qu'on a validée) : propose de l'ajouter, ne le fais pas en cachette.

## Layout

Classement par **où le problème se manifeste** : `infrastructure/` = le
self-host (deploy, compose), `mobile/` = tout le Flutter/iOS/Android,
`third-party/` = les services externes.

```
playbook/
├── README.md                              # Ce fichier (+ index par symptôme ↓)
├── project-structure.md                   # Layout root (apps/packages/ops) + interface Make commune
├── infrastructure/                        # Self-host : Docker, CI/CD, VPS
│   ├── github-actions-deploy.md           # Build → GHCR → SSH → pull ; rollback sans rebuild + retour auto
│   └── docker-compose-overlay.md          # Base env-neutre + overlays dev/prod ; env_file, $$, grace period
├── mobile/                                # Flutter, iOS, Android
│   ├── flutter-mobile-build.md            # Makefile + Fastfile + dart-defines partagés
│   ├── android-release-signing.md         # Keystore + key.properties + Play App Signing (2 SHA !), keytool via JBR
│   ├── ios-push-entitlements.md           # aps-environment dev/prod via 2 fichiers entitlements
│   ├── apple-team-id-and-bundle-id.md     # Team ID + bundle id : TOUJOURS demander à Charles, jamais deviner
│   ├── flutter-inapp-legal-webview.md     # Liens légaux en WebView in-app, pas Safari
│   └── flutter-ios26-liquid-glass-keyboard.md # Coins blancs clavier iOS 26 = resize:false sur TOUTE la chaîne Scaffold
└── third-party/                           # Intégrations SaaS / API externes
    └── revenuecat.md                      # webhook: pas de HMAC, Authorization header
```

## Index par symptôme

Le point d'entrée le plus rapide : tu observes ça → lis cette fiche.

| Tu observes | Fiche |
|---|---|
| Run de deploy vert mais rien n'a changé sur le VPS (backend pas redémarré) | `infrastructure/github-actions-deploy.md` piège #9 (stdin slurp) |
| Un rollback qui déploie du code… récent | `infrastructure/github-actions-deploy.md` (build gardé par `tag == latest`) |
| Backend en crash-loop après une modif du `.env` prod (valeur avec `$`) | `infrastructure/docker-compose-overlay.md` (`$$`) |
| Var présente dans `.env` prod mais absente du process | `infrastructure/docker-compose-overlay.md` (env_file vs environment) |
| Crash prod *"mount a directory onto a file"* | `infrastructure/docker-compose-overlay.md` (bind mount dev qui fuite) |
| `/api/*` renvoie le HTML du SPA | `infrastructure/github-actions-deploy.md` piège #7 |
| Webhooks RevenueCat tous en 401 | `third-party/revenuecat.md` (pas de HMAC — header Authorization statique) |
| Build TestFlight sans paywall / clé RC vide | `mobile/flutter-mobile-build.md` (dart-defines absents du lane Fastlane) |
| Google Sign-In `DEVELOPER_ERROR` en prod Android, OK en debug | `mobile/android-release-signing.md` (2 SHA — Play App Signing) |
| `keytool` : *"Unable to locate a Java Runtime"* | `mobile/android-release-signing.md` (JBR Android Studio) |
| Push OK en `flutter run`, morts en TestFlight | `mobile/ios-push-entitlements.md` (aps-environment) |
| Coins blancs au-dessus du clavier iOS 26 | `mobile/flutter-ios26-liquid-glass-keyboard.md` (resize sur TOUTE la chaîne) |
| Tap "Terms"/"Privacy" éjecte dans Safari | `mobile/flutter-inapp-legal-webview.md` |
| Signature device iOS impossible / app sous le mauvais compte | `mobile/apple-team-id-and-bundle-id.md` (demander, jamais deviner) |
| Tests backend en 401 aléatoires avec dotenv | `mobile/flutter-mobile-build.md` (`\|\|=` vs `??=` sur env vide) |
| `Failed query` sur une colonne qui existe pourtant dans le code | `project-structure.md` (dérive de schéma local — `make db-migrate`) |

Chaque fiche garde sa section « Symptômes à reconnaître » plus détaillée ;
cette table n'est que l'aiguillage. **Une nouvelle fiche = une ligne ici.**

## Règles d'écriture

- **Densité > exhaustivité.** Une fiche de 30 lignes qui dit "voici le piège, voici la bonne façon, voici la doc à jour" bat une fiche de 300 lignes qui paraphrase la doc officielle.
- **Lien vers la doc officielle obligatoire**, parce que la nôtre périme.
- **Le "pourquoi"**, pas que le "quoi". On doit pouvoir juger une edge case en lisant la note, pas juste appliquer une recette à l'aveugle.
- **Date de dernière vérif** en bas de chaque fiche, pour qu'on sache si elle commence à dater.
