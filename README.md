# Coding Playbook

Knowledge base partagée entre **Charles** (l'humain) et **les instances de Claude Code** qui bossent sur ses projets (`listable`, `dian-dian`, etc.).

L'idée : éviter de re-tomber dans les mêmes pièges. Chaque fois qu'on découvre un truc non-évident — bug subtil dans une intégration tierce, gotcha d'un outil, convention qui marche bien sur un projet — on l'écrit ici. La prochaine fois (ou sur la prochaine app), on lit avant de coder.

## Pour les instances Claude Code

Quand on te demande d'intégrer un service tiers (RevenueCat, R2, Resend, Firebase, etc.), un pattern d'infra (Docker Compose, GHCR, deploy), ou une convention propre au workflow de Charles : **vérifie d'abord si une fiche existe ici**. Si oui, applique ce qui est dedans plutôt que de partir des suppositions par défaut. Si la fiche est incomplète ou contredit ton intuition basée sur la doc à jour, signale-le avant de coder.

Quand tu apprends quelque chose qui mérite d'être capturé (bug récurrent, contre-intuitif, ou simplement une convention qu'on a validée) : propose de l'ajouter, ne le fais pas en cachette.

## Layout

```
playbook/
├── README.md                              # Ce fichier
├── third-party/                           # Intégrations SaaS / API externes
│   └── revenuecat.md                      # webhook: pas de HMAC, Authorization header
├── infrastructure/                        # Self-host, Docker, CI/CD, mobile build
│   ├── github-actions-deploy.md           # Build → GHCR → SSH → pull, trigger manuel
│   ├── docker-compose-overlay.md          # Base env-neutre + overlays dev/prod
│   ├── flutter-mobile-build.md            # Makefile + Fastfile + dart-defines partagés
│   ├── flutter-inapp-legal-webview.md     # Liens légaux en WebView in-app, pas Safari
│   ├── flutter-ios26-liquid-glass-keyboard.md # Coins blancs clavier iOS 26 = resize:false sur TOUTE la chaîne Scaffold (shell inclus)
│   └── ios-push-entitlements.md           # aps-environment dev/prod via 2 fichiers entitlements
└── conventions/                           # Conventions code/process (à venir)
```

## Règles d'écriture

- **Densité > exhaustivité.** Une fiche de 30 lignes qui dit "voici le piège, voici la bonne façon, voici la doc à jour" bat une fiche de 300 lignes qui paraphrase la doc officielle.
- **Lien vers la doc officielle obligatoire**, parce que la nôtre périme.
- **Le "pourquoi"**, pas que le "quoi". On doit pouvoir juger une edge case en lisant la note, pas juste appliquer une recette à l'aveugle.
- **Date de dernière vérif** en bas de chaque fiche, pour qu'on sache si elle commence à dater.
