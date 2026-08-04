# Deploy backend self-host — GH Actions + GHCR + trigger manuel

## Le bon shape

```
[laptop] → ./ops/scripts/deploy.sh        # appelle gh workflow run
   ↓
[GitHub Actions runner]
   ├─ build Docker image
   ├─ push ghcr.io/<owner>/<repo>-backend:<short-sha>
   ├─ push ghcr.io/<owner>/<repo>-backend:latest
   ├─ ssh into VPS
   ├─ rsync compose files + Caddyfile only (PAS le source code)
   ├─ docker compose pull backend
   ├─ docker compose up -d --wait postgres
   ├─ docker compose run --rm --no-deps backend <migrate>  (3× retry)
   └─ docker compose up -d --wait
```

## Ce qu'on évite (et pourquoi)

| Anti-pattern | Pourquoi non |
|---|---|
| Build sur le VPS (rsync source + `docker compose build`) | Sature un petit VPS (2 vCPU) pendant le build, fait race condition postgres ↔ drizzle-kit sur `pg_filenode.map`, oblige `rsync --delete` qui peut wipe les data dirs si une exclusion est oubliée |
| Auto-deploy sur `push master` | Pas de garde-fou avant prod ; un typo dans un commit est en prod en 3 minutes ; pas de rollback sans `git revert` |
| Watchtower / agent qui pull tout seul | Boîte noire pour le débogage, runs aléatoires, pas de log centralisé du run |
| Fine-grained PAT pour pull GHCR | **Bug GitHub connu** : GHCR ne supporte pas les fine-grained PATs, seulement les classic. C'est documenté nulle part proprement, ça te fait perdre 30 min |

## Workflow file (généralisable)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Tag to deploy (latest = HEAD short SHA, or pass a 7-char SHA to roll back)"
        required: false
        default: latest

concurrency:
  group: deploy
  cancel-in-progress: false

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
      image_name: ${{ steps.meta.outputs.image_name }}
    steps:
      - uses: actions/checkout@v4

      - id: meta
        env:
          OWNER: ${{ github.repository_owner }}
          INPUT_TAG: ${{ github.event.inputs.tag }}
        run: |
          # 'latest' = build HEAD ; un sha7 = ROLLBACK vers une image EXISTANTE.
          # Valider l'input : un tag arbitraire ne doit jamais atteindre le shell.
          if ! [[ "$INPUT_TAG" =~ ^([0-9a-f]{7}|latest)$ ]]; then
            echo "tag invalide: '$INPUT_TAG' (attendu: 'latest' ou un SHA court de 7 hex)"; exit 1
          fi
          if [ "$INPUT_TAG" = "latest" ]; then
            echo "tag=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
          else
            echo "tag=$INPUT_TAG" >> "$GITHUB_OUTPUT"
          fi
          owner_lc="${OWNER,,}"  # GHCR exige lowercase
          echo "image_name=ghcr.io/${owner_lc}/<repo>-backend" >> "$GITHUB_OUTPUT"

      # ⚠️ Les 3 steps de build sont GARDÉS par `tag == 'latest'` : un rollback
      # ne rebuild JAMAIS. Sans ces if, un rollback vers abc1234 rebuilderait
      # le code HEAD et l'ÉCRASERAIT sous le tag abc1234 sur GHCR — ton
      # "rollback" déploierait du code neuf déguisé en vieux SHA.
      - uses: docker/setup-buildx-action@v3
        if: github.event.inputs.tag == 'latest'
      - uses: docker/login-action@v3
        if: github.event.inputs.tag == 'latest'
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        if: github.event.inputs.tag == 'latest'
        with:
          context: .
          file: apps/backend/Dockerfile
          target: runtime
          push: true
          tags: |
            ${{ steps.meta.outputs.image_name }}:${{ steps.meta.outputs.tag }}
            ${{ steps.meta.outputs.image_name }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.VPS_SSH_KEY }}" > ~/.ssh/id_deploy
          chmod 600 ~/.ssh/id_deploy
          ssh-keyscan -H ${{ secrets.VPS_HOST }} >> ~/.ssh/known_hosts

      - name: Sync compose + Caddyfile
        env:
          VPS_HOST: ${{ secrets.VPS_HOST }}
          VPS_USER: ${{ secrets.VPS_USER }}
        run: |
          # Pas de --delete pour éviter le piège data-dir.
          ssh -i ~/.ssh/id_deploy "$VPS_USER@$VPS_HOST" 'mkdir -p /srv/<app>/ops'
          # Tout overlay prod actif voyage ICI **et** doit être dans $COMPOSE
          # plus bas (piège #12) — avec son contexte de build et les scripts
          # appelés par le cron hôte.
          rsync -az -e 'ssh -i ~/.ssh/id_deploy' \
            ops/docker-compose.yml \
            ops/docker-compose.prod.yml \
            "$VPS_USER@$VPS_HOST:/srv/<app>/ops/"

          # Caddyfile en place via cat > (préserve l'inode pour le
          # bind mount Caddy, sinon le container voit toujours l'ancien).
          scp -i ~/.ssh/id_deploy ops/Caddyfile \
            "$VPS_USER@$VPS_HOST:/tmp/<app>-Caddyfile"
          ssh -i ~/.ssh/id_deploy "$VPS_USER@$VPS_HOST" \
            "cat /tmp/<app>-Caddyfile > /srv/<app>/ops/Caddyfile && rm /tmp/<app>-Caddyfile"

      - name: Pull image, migrate, restart
        env:
          VPS_HOST: ${{ secrets.VPS_HOST }}
          VPS_USER: ${{ secrets.VPS_USER }}
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image_tag }}
        run: |
          # Passage de IMAGE_TAG via env inline ssh + heredoc quoted
          # pour éviter les double-expansions.
          ssh -i ~/.ssh/id_deploy "$VPS_USER@$VPS_HOST" \
            "IMAGE_TAG='$IMAGE_TAG' bash -s" << 'REMOTE'
          set -euo pipefail
          cd /srv/<app>
          # dernier tag connu-bon : persisté dans .env APRÈS un déploiement
          # réussi seulement → cible de retour si la nouvelle image est
          # mauvaise, et fige les `docker compose up -d` manuels sur ce tag.
          PREV="$(grep -E '^IMAGE_TAG=' .env | head -1 | cut -d= -f2- || true)"

          COMPOSE="docker compose --env-file /srv/<app>/.env \
            -f ops/docker-compose.yml \
            -f ops/docker-compose.prod.yml"

          # échoue ici si le tag n'existe pas sur GHCR (rollback vers un tag inconnu)
          $COMPOSE pull backend
          $COMPOSE up -d --wait postgres

          # Valider le nouveau Caddyfile AVANT de recréer quoi que ce soit.
          # </dev/null OBLIGATOIRE : ce script arrive par stdin (bash -s <<
          # heredoc) et `docker exec -T` attache stdin → il AVALERAIT tout le
          # reste du script, qui « réussirait » sans jamais redémarrer le
          # backend (cf. piège #9 — vécu).
          if [ -n "$($COMPOSE ps -q caddy 2>/dev/null)" ]; then
            $COMPOSE exec -T caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile </dev/null
          fi

          # Migrate avec retry x3 pour la race pg_filenode.map (rare
          # mais intrinsèque à postgres DDL).
          for attempt in 1 2 3; do
            if $COMPOSE run --rm --no-deps -T backend <migrate-cmd> </dev/null; then
              break
            fi
            if [ $attempt -eq 3 ]; then
              echo "migrate failed 3 times, aborting"
              exit 1
            fi
            sleep 5
          done

          if ! $COMPOSE up -d --wait; then
            echo "nouvelle image unhealthy"
            if [ -n "$PREV" ] && [ "$PREV" != "$IMAGE_TAG" ]; then
              echo "retour automatique au tag connu-bon $PREV"
              IMAGE_TAG="$PREV" $COMPOSE up -d --wait || echo "retour à $PREV échoué — intervention manuelle requise"
            fi
            exit 1
          fi

          # succès → persister le tag connu-bon dans .env (idempotent)
          if grep -qE '^IMAGE_TAG=' .env; then
            sed -i "s|^IMAGE_TAG=.*|IMAGE_TAG=$IMAGE_TAG|" .env
          else
            printf '\nIMAGE_TAG=%s\n' "$IMAGE_TAG" >> .env
          fi

          # Recharge Caddy en place (--adapter + --force : cf. piège #8).
          $COMPOSE exec -T caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile --force </dev/null

          docker image prune -f
          # Sentinelle : si un exec a avalé la fin du script, cette ligne
          # n'apparaît pas dans les logs → la troncature devient VISIBLE.
          echo "REMOTE_SCRIPT_COMPLET (tag $IMAGE_TAG)"
          REMOTE
```

Le compose prod doit référencer `image: ghcr.io/<owner>/<repo>-backend:${IMAGE_TAG:-latest}`
pour que le tag persisté dans `/srv/<app>/.env` pilote pull ET up (cf.
`docker-compose-overlay.md`).

## Script wrapper côté laptop

```bash
# ops/scripts/deploy.sh
#!/usr/bin/env bash
set -euo pipefail

tag="${1:-latest}"

if ! command -v gh >/dev/null 2>&1; then
  echo "gh CLI missing — brew install gh" >&2
  exit 1
fi

echo "==> Trigger deploy (tag=$tag)..."
gh workflow run deploy.yml -f tag="$tag"
sleep 3
echo "==> Tailing run..."
gh run watch
```

Usage :
- `./ops/scripts/deploy.sh` → deploy HEAD
- `./ops/scripts/deploy.sh abc1234` → rollback à un tag précédent (l'image doit toujours exister sur GHCR)

> **Porte d'entrée laptop** : on wrappe le script dans le `Makefile` racine —
> `make deploy` (= `deploy.sh`), `make deploy TAG=<sha7>` pour un rollback. Le
> déclenchement reste **manuel** : script **ou** bouton *Run workflow* dans
> l'onglet Actions — jamais `on: push` (cf. anti-patterns).

## Front statique (Vite/SPA) : un deploy SÉPARÉ du backend

Quand le backend sert le SPA en prod, on le sert plutôt via **Caddy**
(`file_server`) et on déploie le front **indépendamment** : un changement de
front ne doit jamais rebuild/redémarrer le backend (ni couper les bots /
WebSockets en cours). Deux pipelines `workflow_dispatch` distincts :

- **Backend** : `make deploy` → build image → GHCR → pull/migrate/restart (ci-dessus).
- **Front** : `make deploy-web` → build le SPA + `rsync` du `dist/` vers le dossier
  servi par Caddy. **Aucun contact avec le backend** → zéro interruption du live.

```yaml
# .github/workflows/deploy-web.yml
name: Deploy Web
on: { workflow_dispatch: {} }
concurrency: { group: deploy-web, cancel-in-progress: true }
jobs:
  build-and-sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: 1.2.x }
      - run: bun install --frozen-lockfile
      - run: bun run --cwd apps/web build
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.VPS_SSH_KEY }}" > ~/.ssh/id_deploy
          chmod 600 ~/.ssh/id_deploy
          ssh-keyscan -H ${{ secrets.VPS_HOST }} >> ~/.ssh/known_hosts
      - name: Rsync dist → VPS (front only)
        env: { VPS_HOST: "${{ secrets.VPS_HOST }}", VPS_USER: "${{ secrets.VPS_USER }}" }
        run: |
          ssh -i ~/.ssh/id_deploy "$VPS_USER@$VPS_HOST" 'mkdir -p /srv/<app>/web'
          # --delete OK ICI : cible = le sous-dossier web/ UNIQUEMENT (jamais /srv/<app>)
          rsync -az --delete -e 'ssh -i ~/.ssh/id_deploy' \
            apps/web/dist/ "$VPS_USER@$VPS_HOST:/srv/<app>/web/"
```

Le `deploy-web.sh` côté laptop est le miroir de `deploy.sh` (`gh workflow run
deploy-web.yml` + `gh run watch`). Côté Caddy, l'API et le SPA cohabitent —
**router l'API/WS AVANT** le fallback SPA (cf. pièges #7) :

```caddyfile
{$DOMAIN} {
    @backend path /api/* /ws /ws/*
    handle @backend { reverse_proxy backend:3001 }
    handle {                         # tout le reste = SPA
        root * /srv/www
        try_files {path} /index.html
        file_server
    }
}
```

## Setup avant le premier deploy

**Côté GitHub (depuis le laptop, dans le repo) :**
```bash
gh secret set VPS_HOST   --body '<vps-ip>'
gh secret set VPS_USER   --body 'root'
gh secret set VPS_SSH_KEY < ~/.ssh/<deploy-key-privée>   # le `<` est crucial, pas un arg positionnel
```

**Côté VPS (une fois) :**
1. Génère un PAT GitHub **classic** (PAS fine-grained) sur https://github.com/settings/tokens
   - Nom : `<app>-ghcr-pull`
   - Expiration : 1 year (oblige rotation), pas "no expiration"
   - Scope : **uniquement `read:packages`** — rien d'autre
2. ```bash
   ssh root@<vps>
   echo '<PAT>' | docker login ghcr.io -u <github-user> --password-stdin
   ```
   La cred est stockée dans `~/.docker/config.json` (en clair, base64). Le blast radius en cas de fuite VPS est limité à "lire tes packages privés" → c'est pour ça qu'on prend un scope minimal.
3. Crée les volumes runtime que la prod compose attend :
   ```bash
   mkdir -p /srv/<app>/{postgres,caddy/data,caddy/config,ops}
   chown -R 999:999 /srv/<app>/postgres  # postgres uid dans l'image alpine
   ```
   **Règle générale** : chaque bind mount **écrit** doit appartenir à l'uid du
   process dans l'image. Si le backend tourne non-root (`USER bun` → uid 1000,
   bonne pratique), son dossier data doit être `chown 1000:1000` — sinon
   permission denied au premier write, parfois longtemps après le deploy.
4. Crée `/srv/<app>/.env` mode 0600, root-only, avec tous les secrets prod (cf. ton runbook). ⚠️ Tout `$` littéral dans une valeur s'écrit `$$` (piège #11).

## Rollback

```bash
./ops/scripts/deploy.sh <short-sha>
```

Trois propriétés que le workflow ci-dessus garantit (et qu'un rollback naïf n'a pas) :

1. **Jamais de rebuild** : les steps de build sont gardés par `tag == 'latest'`.
   Le rollback pull une image **existante** sur GHCR — si elle n'existe pas, le
   `pull` échoue proprement. Browse : `https://github.com/<owner>/<repo>/pkgs/container/<repo>-backend`.
2. **Retour automatique** : si la nouvelle image ne devient pas healthy
   (`up -d --wait` échoue), le script revient au dernier tag connu-bon (`IMAGE_TAG`
   de `/srv/<app>/.env`) tout seul.
3. **Le connu-bon survit aux ops manuelles** : `IMAGE_TAG` étant persisté dans le
   `.env` du VPS, un `docker compose up -d` fait à la main sur le VPS reste sur
   le tag déployé — pas de dérive vers `:latest`.

Si la migration de cet ancien SHA inclut une DDL irréversible (`DROP COLUMN`, etc.), le rollback ne suffit pas — il faut une restore de DB séparée. Drizzle est forward-only par design.

## Pièges déjà rencontrés

1. **Heredoc SSH + variables locales** : si tu mets `<< REMOTE` (non quoted), bash expand les `$VAR` localement avant d'envoyer. Si `<< 'REMOTE'` (quoted), tout est envoyé verbatim — préférable. Pour passer une variable du runner au remote, l'inline avec `ssh ... "VAR='$VAR' bash -s" << 'REMOTE'` est le pattern le plus lisible.

2. **`--delete` rsync data wipe** : si tu rsync `./` vers `/srv/<app>/` avec `--delete`, tout ce qui n'est pas dans le repo (postgres data dir, caddy volumes, .env) disparaît. Le nouveau setup envoie uniquement les compose files par chemin explicite — pas de `--delete` du tout.

3. **Caddy bind mount inode** : les bind mounts capturent l'inode du fichier au démarrage du container. Si rsync remplace le fichier (nouvel inode), le container voit toujours l'ancien jusqu'à un restart. Solution : `cat new > old` qui modifie le fichier en place, ou recréer le container Caddy.

4. **GHCR + fine-grained PAT** : ne marche pas. Documentation officielle GitHub : "Fine-grained personal access tokens cannot currently be used to manage Container Registry packages." Toujours utiliser un classic PAT avec scope `read:packages`.

5. **`docker compose pull` ignore les changements de compose.yml** : si le tag image est dans une variable d'env (`${IMAGE_TAG}`), le `pull` honore la nouvelle valeur. Mais si tu changes le `image:` directement dans le yml et ne re-rsync pas, le VPS continue de pull l'ancien. Toujours rsync les compose avant le pull.

6. **`up -d` vs `up -d --wait`** : sans `--wait`, le workflow rend la main dès que les containers sont "started" même si leur healthcheck rate. Avec `--wait`, ça bloque jusqu'à healthy. Toujours `--wait` en CI pour que les runs verts garantissent un service réellement UP.

7. **API avalée par le SPA** : avec `file_server` + `try_files {path} /index.html`, les requêtes `/api/*` sont réécrites vers `index.html` AVANT d'atteindre le `reverse_proxy`. Router l'API dans un `handle @backend { reverse_proxy … }` placé AVANT le `handle { … file_server }` du SPA (snippet de la section front).

8. **Caddy `reload` qui n'applique pas** : `caddy reload --config /etc/caddy/Caddyfile` peut sortir en exit 0 sans recharger le nouveau Caddyfile. Forcer l'adaptateur + le reload : `--adapter caddyfile --force`. Avec le piège #3 (inode du bind mount), c'est ce qui garantit qu'un changement de Caddyfile prend effet sans recréer le container.

9. **`docker exec -T` avale le script `bash -s` (stdin slurp)** : le script remote arrive **par stdin** (`ssh … bash -s << 'REMOTE'`). Tout `docker compose exec -T` / `run -T` au milieu attache stdin au conteneur et **aspire le reste du heredoc** — bash atteint EOF et sort en **exit 0**. Symptôme vécu : run Actions vert, sentinelle absente des logs, backend jamais redémarré, `IMAGE_TAG` jamais persisté. Parade double : `</dev/null` sur **chaque** exec/run du heredoc, + une **sentinelle** `echo REMOTE_SCRIPT_COMPLET` en dernière ligne (une fin avalée devient visible dans les logs au lieu de passer pour un succès).

10. **`set -euo pipefail` + pipeline légitimement vide** : un pattern du genre `( crontab -l | grep -v 'marqueur' ; echo '<nouvelle ligne>' ) | crontab -` (cron idempotent) **explose au premier run** — sans crontab existante (ou sans autre ligne que le marqueur), `crontab -l | grep -v` sort non-zéro et pipefail tue le script. Ajouter `|| true` sur le pipeline de lecture : `( crontab -l 2>/dev/null | grep -v 'marqueur' || true ; echo '…' ) | crontab -`. Généralisable : tout `grep`/`ls` d'un état « peut-être vide » dans un script pipefail doit être toléré explicitement.

11. **`$` littéral dans `/srv/<app>/.env`** : compose interpole les valeurs d'`env_file` — un secret contenant `$` (mot de passe, hash) est tronqué/vidé silencieusement. L'écrire `$$`. Détails et symptôme vécu dans `docker-compose-overlay.md` (section env_file).

12. **Un overlay rsyncé mais absent des commandes `compose` (vécu 2026-08, backups tués en silence)** : dès qu'une fonctionnalité prod vit dans un overlay opt-in (`docker-compose.prod-backup.yml` pour l'archivage WAL, par exemple), il ne suffit **pas** de l'envoyer sur le VPS. S'il n'est pas dans le `-f` de la variable `$COMPOSE`, le `up -d --wait postgres` du deploy **recrée le service depuis l'overlay de base** — image vanilla, `archive_mode` perdu. Rien n'échoue : le deploy est vert, la DB tourne, les backups continuent même à « réussir »… et on l'apprend au premier restore. Règle : **tout overlay actif en prod appartient à `$COMPOSE`**, et son `--env-file` avec.
    ```bash
    COMPOSE="docker compose \
      --env-file /srv/<app>/.env \
      --env-file /srv/<app>/.env.pgbackrest \
      -f ops/docker-compose.yml \
      -f ops/docker-compose.prod.yml \
      -f ops/docker-compose.prod-backup.yml"
    ```
    Corollaire souvent oublié : si l'overlay contient un `build:`, le **contexte de build** doit voyager avec (`rsync ops/pgbackrest`), sinon une box fraîche ou un `docker image prune` casse le deploy — et les changements de config (`pgbackrest.conf`) n'atteignent jamais le VPS. Même chose pour les scripts appelés par le cron hôte (`rsync ops/scripts` + `chmod +x`).
    **Vérif** : `docker compose <tous les -f> config | grep -E "image:|archive_mode"` doit montrer l'image et les options attendues.

13. **Recréer un service casse les pools de ceux qui en dépendent — et `/healthz` ne le voit pas (vécu 2026-08)** : recréer le conteneur postgres laisse le backend avec des connexions mortes dans son pool. Il répond **200 à `/healthz`** (le process vit, il sert du HTTP) pendant que la base est injoignable. Seul `/readyz` bascule à `{"ready":false,"checks":{"database":false}}`. C'est la distinction **liveness vs readiness** : `healthz` répond « le process tourne », `readyz` répond « je peux servir une requête réelle ». Un monitoring qui ne surveille que le premier voit du vert pendant une panne totale.
    - **Exposer les deux**, et faire porter l'alerting sur `readyz`.
    - Après tout `up -d`/`restart` d'un service **dont d'autres dépendent**, enchaîner par un `up -d` des dépendants, puis vérifier `readyz` — pas `healthz`.
    - `up -d --wait` ne sauve pas : il attend le healthcheck **du conteneur recréé**, pas la reconvergence de ses consommateurs.

## Quand passer à autre chose

Ce pattern tient jusqu'à ~10K users et un seul VPS. Au-delà :
- Multi-instance backend → load balancer en front, et il faut un message bus (Postgres LISTEN/NOTIFY ou NATS) pour la broadcast WebSocket cross-process
- Postgres single-primary devient le bottleneck → read replica
- Compose devient pénible → migration k8s (Talos / k0s sur le même VPS, ou managed type DOKS / GKE)

Mais pour solo-dev pré-PMF, ce stack est optimal.

---

**Dernière vérif** : 2026-08-04 (ajout du piège #12 — overlay backup rsyncé mais
absent de `$COMPOSE`, trouvé en comparant deux déploiements : le prochain deploy
aurait recréé postgres sans archivage WAL, sans aucun signal).
