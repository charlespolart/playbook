# Structure repo — layout root & interface de lancement

## Le problème qu'on fige

D'un projet à l'autre, le lancement divergeait (`bun run dev`, `bun run dev:web`,
`make app`, `cd apps/backend && bun run dev`…) et les dossiers aussi (`backend/` +
`flutter_app/` vs `apps/`, `docker-compose.yml` à la racine vs `ops/`). Résultat :
aucune mémoire musculaire transférable, et plusieurs portes d'entrée pour la même
action. On fige donc **la structure root** et **l'interface Make** — **pas la
techno** (elle dépend du projet : web/mobile/perf/contraintes).

## Ce qui est figé (1) — layout root

```
<repo>/
  apps/            # apps déployables, 1 dossier par rôle
    backend/       #   l'API / le service
    web/           #   front web        (si applicable)
    mobile/        #   app Flutter      (si applicable)
  packages/        # code partagé entre apps (libs internes)
  ops/             # infra & déploiement : docker-compose, scripts deploy  → docker-compose-overlay.md
  docs/
  tools/           # (optionnel) scripts dev / CI
  Makefile         # interface de commandes unique  →  make help
  README.md  CLAUDE.md
  <manifeste racine selon la stack : package.json workspace, melos.yaml, go.work…>
```

Règles (indépendantes de la techno) :
1. **Toute app déployable vit dans `apps/<rôle>`**, jamais à la racine. Noms par
   rôle : `backend`, `web`, `mobile` — pas `server`, `flutter_app`, etc.
2. **Code partagé → `packages/`** (1 dossier par lib).
3. **Infra/déploiement → `ops/`** (compose + scripts). Pas de compose à la racine.
4. **Un `.env` par app** (`apps/<x>/.env` + `.env.example` tracké). Pas de `.env`
   partagé référencé en `../../` (sinon couplage de chemins + secrets mobile/back
   mélangés — cf. la distinction mobile vs backend dans `flutter-mobile-build.md`).
5. **Un `Makefile` racine** = unique porte d'entrée.

Un projet n'a pas forcément les 3 apps : web pur = `apps/{backend,web}` ; mobile =
`apps/{backend,mobile}`. La structure root reste identique.

## Ce qui est figé (2) — interface Make

Le Makefile est l'**unique interface** sur tous les projets (même un web simple).
Il **délègue** aux commandes réelles de la stack → l'interface reste identique
même quand la techno change. Verbes communs :

| Cible | Rôle | back | web | mobile |
|---|---|:-:|:-:|:-:|
| `help` (défaut) | liste auto des commandes | ✅ | ✅ | ✅ |
| `back` | backend en dev | ✅ | ✅ | ✅ |
| `app` | front mobile (`flutter run`) | | | ✅ |
| `app-web` | front mobile dans Chrome | | | ✅ |
| `web` | front web en dev (vite…) | | ✅ | |
| `db` / `db-stop` | base de données (docker) | ✅ | ✅ | ✅ |
| `deploy` | déploie le backend (CI → GHCR → VPS, manuel) → `github-actions-deploy.md` | ✅ | ✅ | |
| `deploy-web` | déploie le front statique (rsync, séparé du back) | | ✅ | |
| `ipa` / `apk` | builds stores | | | ✅ |

**Conventions Make générales (tous projets) :**
- Recettes indentées par **TABULATIONS** (sinon `make` casse : "missing separator").
- `help` **auto-documenté** : chaque cible porte un `## description` que `help` grep ;
  `.DEFAULT_GOAL := help` → `make` seul liste tout.
- `db`/`db-stop` passent par l'**overlay dev** (expose le port hôte, sinon un back
  natif ne joint pas la DB dockerisée) → `infrastructure/docker-compose-overlay.md`.

**Socle commun** = `back` / `db` / `db-stop` / `help`. Gabarit **web** (y ajoute `web` / `build`) :

```make
.PHONY: help back web db db-stop build deploy deploy-web
.DEFAULT_GOAL := help
# back natif contre Postgres dockerisé → overlay dev expose le port hôte (cf. docker-compose-overlay.md)
COMPOSE_DEV := docker compose -f ops/docker-compose.yml -f ops/docker-compose.dev.yml

back:    ## Backend dev
	bun run --cwd apps/backend dev
web:     ## Front web dev
	bun run --cwd apps/web dev
db:      ## DB (docker, arrière-plan)
	$(COMPOSE_DEV) up -d postgres
db-stop: ## Stoppe la DB
	$(COMPOSE_DEV) down
build:   ## Build du front
	bun run --cwd apps/web build
deploy:     ## Déploie le backend (CI → GHCR → VPS, manuel). Rollback : make deploy TAG=<sha7>
	./ops/scripts/deploy.sh $(TAG)
deploy-web: ## Déploie le front (build + rsync, n'impacte PAS le backend)
	./ops/scripts/deploy-web.sh
help:    ## Liste les commandes
	@grep -E '^[a-z-]+:.*?##' $(MAKEFILE_LIST) | awk 'BEGIN{FS=":.*?## "}; {printf "  \033[36m%-10s\033[0m %s\n", $$1, $$2}'
```

**Mobile** : ajoute les cibles `app` / `app-web` / `ipa` / `apk` (injection des
`--dart-define`) — script + détails dans `infrastructure/flutter-mobile-build.md`.

## Ce qui n'est PAS figé

La **techno**. Bun/Vite/Flutter/Postgres dans les exemples ne sont **que des
exemples** — choisis selon le projet. Seuls le **layout root** et les **verbes Make**
sont imposés. Le Makefile traduit le verbe vers ce que le projet utilise réellement.

## Organisation interne d'une app (indicatif, à adapter)

- `apps/backend/src/` : `routes/` (transport), `services/` (I/O externe : email,
  push, paiements), `db/` (client+schema+migrations), `middleware/`, `lib/` (pur).
  Optionnels : `ws/`, `queues/`, `observability/`. Logique métier réutilisable →
  `packages/core`.
- `apps/web/src/` : `pages/`, `components/`, `lib/` (+ `hooks/`, `api/`, `stores/`).
- `apps/mobile/lib/src/` : `app/ router/ providers/ screens/ widgets/ services/
  models/ i18n/ util/`. Quand ça grossit → feature-first dans `packages/` (melos) :
  `packages/{core, data, ui_kit, features/<feature>}`.
- `packages/` : `core` (domaine pur), `shared` (types front↔back), `data`
  (repos/sources), `db` (schéma+migrations), `ui_kit` (design system Flutter).

## Checklist nouveau projet

1. `apps/{backend[,web][,mobile]}` + `packages/` (si partage) + `ops/` + `docs/`.
2. Un `.env.example` par app.
3. DB en docker dans `ops/` (base + overlay dev → `docker-compose-overlay.md`).
4. `Makefile` racine — gabarit web ci-dessus, ou `flutter-mobile-build.md` pour mobile.
5. `make help` liste `back` / (`app`|`web`) / `db` dès le départ.

---

**Dernière vérif** : 2026-06-28 (ajout cibles `deploy` / `deploy-web`).
**Voir aussi** : `infrastructure/flutter-mobile-build.md` (Makefile mobile + secrets), `infrastructure/docker-compose-overlay.md` (`ops/`).
