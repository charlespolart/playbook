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
        run: |
          if [ "${{ github.event.inputs.tag }}" = "latest" ]; then
            echo "tag=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
          else
            echo "tag=${{ github.event.inputs.tag }}" >> $GITHUB_OUTPUT
          fi
          owner_lc="${OWNER,,}"  # GHCR exige lowercase
          echo "image_name=ghcr.io/${owner_lc}/<repo>-backend" >> $GITHUB_OUTPUT
        env:
          OWNER: ${{ github.repository_owner }}

      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
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
          rsync -az -e 'ssh -i ~/.ssh/id_deploy' \
            ops/docker-compose.yml \
            ops/docker-compose.prod.yml \
            "$VPS_USER@$VPS_HOST:/srv/<app>/ops/"

          # Caddyfile en place via cat > (préserve l'inode pour le
          # bind mount Caddy, sinon le container voit toujours l'ancien).
          scp -i ~/.ssh/id_deploy ops/Caddyfile \
            "$VPS_USER@$VPS_HOST:/tmp/<app>-Caddyfile"
          ssh -i ~/.ssh/id_deploy "$VPS_USER@$VPS_HOST" \
            "mkdir -p /srv/<app>/ops && cat /tmp/<app>-Caddyfile > /srv/<app>/ops/Caddyfile && rm /tmp/<app>-Caddyfile"

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
          echo "$IMAGE_TAG" > .deployed-tag

          COMPOSE="docker compose --env-file /srv/<app>/.env \
            -f ops/docker-compose.yml \
            -f ops/docker-compose.prod.yml"

          $COMPOSE pull backend
          $COMPOSE up -d --wait postgres

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

          $COMPOSE up -d --wait

          # Recharge Caddy en place (Caddyfile peut avoir changé).
          $COMPOSE exec -T caddy caddy validate --config /etc/caddy/Caddyfile
          $COMPOSE exec -T caddy caddy reload --config /etc/caddy/Caddyfile

          docker image prune -f
          REMOTE
```

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
4. Crée `/srv/<app>/.env` mode 0600, root-only, avec tous les secrets prod (cf. ton runbook).

## Rollback

```bash
./ops/scripts/deploy.sh <short-sha>
```

L'image avec ce tag doit encore exister sur GHCR. Browse : `https://github.com/<owner>/<repo>/pkgs/container/<repo>-backend`.

Si la migration de cet ancien SHA inclut une DDL irréversible (`DROP COLUMN`, etc.), le rollback ne suffit pas — il faut une restore de DB séparée. Drizzle est forward-only par design.

## Pièges déjà rencontrés

1. **Heredoc SSH + variables locales** : si tu mets `<< REMOTE` (non quoted), bash expand les `$VAR` localement avant d'envoyer. Si `<< 'REMOTE'` (quoted), tout est envoyé verbatim — préférable. Pour passer une variable du runner au remote, l'inline avec `ssh ... "VAR='$VAR' bash -s" << 'REMOTE'` est le pattern le plus lisible.

2. **`--delete` rsync data wipe** : si tu rsync `./` vers `/srv/<app>/` avec `--delete`, tout ce qui n'est pas dans le repo (postgres data dir, caddy volumes, .env) disparaît. Le nouveau setup envoie uniquement les compose files par chemin explicite — pas de `--delete` du tout.

3. **Caddy bind mount inode** : les bind mounts capturent l'inode du fichier au démarrage du container. Si rsync remplace le fichier (nouvel inode), le container voit toujours l'ancien jusqu'à un restart. Solution : `cat new > old` qui modifie le fichier en place, ou recréer le container Caddy.

4. **GHCR + fine-grained PAT** : ne marche pas. Documentation officielle GitHub : "Fine-grained personal access tokens cannot currently be used to manage Container Registry packages." Toujours utiliser un classic PAT avec scope `read:packages`.

5. **`docker compose pull` ignore les changements de compose.yml** : si le tag image est dans une variable d'env (`${IMAGE_TAG}`), le `pull` honore la nouvelle valeur. Mais si tu changes le `image:` directement dans le yml et ne re-rsync pas, le VPS continue de pull l'ancien. Toujours rsync les compose avant le pull.

6. **`up -d` vs `up -d --wait`** : sans `--wait`, le workflow rend la main dès que les containers sont "started" même si leur healthcheck rate. Avec `--wait`, ça bloque jusqu'à healthy. Toujours `--wait` en CI pour que les runs verts garantissent un service réellement UP.

## Quand passer à autre chose

Ce pattern tient jusqu'à ~10K users et un seul VPS. Au-delà :
- Multi-instance backend → load balancer en front, et il faut un message bus (Postgres LISTEN/NOTIFY ou NATS) pour la broadcast WebSocket cross-process
- Postgres single-primary devient le bottleneck → read replica
- Compose devient pénible → migration k8s (Talos / k0s sur le même VPS, ou managed type DOKS / GKE)

Mais pour solo-dev pré-PMF, ce stack est optimal.

---

**Dernière vérif** : 2026-05-13 (en prod).
