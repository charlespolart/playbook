# Docker Compose — split base env-neutre + overlays dev/prod

## La règle d'or

**La base ne doit avoir aucun élément env-spécifique**. Chaque overlay (dev / prod) ne fait que **ajouter**. Si un overlay doit *retirer* quelque chose de la base, le split est mal foutu et tu vas finir avec des `!reset` partout.

## Pourquoi pas l'inverse (base = dev, prod override)

Le piège : Docker Compose **append** les listes (`volumes:`, `ports:`, `env_file:`) lors d'un merge — il ne remplace pas. Donc si ta base a :

```yaml
backend:
  volumes:
    - ../apps/backend/src:/app/apps/backend/src   # hot reload dev
```

…et que ton overlay prod a `volumes: []`, le container prod hérite quand même du bind mount dev. Résultat : crash en prod avec *"Are you trying to mount a directory onto a file?"* parce que `/srv/<app>/apps/backend/src` n'existe pas sur le VPS.

La solution `!reset []` existe mais c'est une rustine. Le bon design rend le `!reset` inutile.

## Shape final

### `ops/docker-compose.yml` — base env-neutre

Pas d'`image:`, pas de `build:`, pas de `volumes:`, pas de `ports:`. Juste ce qui est identique en dev et en prod : topologie, healthcheck, dépendances, tuning postgres, variables d'env partagées (PORT, DATABASE_URL).

```yaml
name: <app>

services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-<app>}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-<app>}
      POSTGRES_DB: ${POSTGRES_DB:-<app>}
      PGDATA: /var/lib/postgresql/data/pgdata
    healthcheck:
      # Hit pg_class pour forcer postgres à passer par le filenode map —
      # `pg_isready` et `SELECT 1` peuvent rendre healthy même quand le
      # catalogue est temporairement inaccessible.
      test:
        - "CMD-SHELL"
        - "psql -U ${POSTGRES_USER:-<app>} -d ${POSTGRES_DB:-<app>} -tAc 'SELECT 1 FROM pg_class LIMIT 1' >/dev/null"
      interval: 3s
      timeout: 5s
      retries: 30
      start_period: 30s
    command:
      - postgres
      - -c
      - log_statement=none
      - -c
      - max_connections=200
      # pg_stat_statements ship avec postgres core, preload-le en dev
      # aussi pour que les dev containers soient shaped comme la prod.
      - -c
      - shared_preload_libraries=pg_stat_statements
      - -c
      - pg_stat_statements.track=all

  backend:
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      PORT: 3000
      DATABASE_URL: postgres://${POSTGRES_USER:-<app>}:${POSTGRES_PASSWORD:-<app>}@postgres:5432/${POSTGRES_DB:-<app>}
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/healthz || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s
```

La base seule n'est pas runnable (backend a ni `image` ni `build`). C'est intentionnel — ça force à toujours layer un overlay.

### `ops/docker-compose.dev.yml`

Ajoute build target=development, ports loopback, hot-reload bind mounts, named volume pour pg, dev env defaults.

```yaml
services:
  postgres:
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  backend:
    build:
      context: ..
      dockerfile: apps/backend/Dockerfile
      target: development
    environment:
      NODE_ENV: development
      LOG_LEVEL: ${LOG_LEVEL:-debug}
      BETTER_AUTH_SECRET: ${BETTER_AUTH_SECRET:-dev-secret-change-me}
      # ... autres dev defaults
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      # node_modules reste dans l'image, on bind-mount uniquement le code.
      - ../apps/backend/src:/app/apps/backend/src:delegated
      - ../apps/backend/drizzle.config.ts:/app/apps/backend/drizzle.config.ts:delegated

volumes:
  postgres_data:
    name: <app>_postgres_data
```

### `ops/docker-compose.prod.yml`

Ajoute image GHCR, bind volumes vers `/srv`, `env_file:` pour les secrets prod, Caddy comme reverse proxy.

```yaml
services:
  postgres:
    restart: always
    volumes:
      - /srv/<app>/postgres:/var/lib/postgresql/data

  backend:
    image: ghcr.io/<owner>/<app>-backend:${IMAGE_TAG:-latest}
    restart: always
    # env_file pousse TOUS les secrets de /srv/<app>/.env dans le
    # container. C'est ce qui évite de devoir lister chaque var
    # dans `environment:` avec `${VAR}` qui rate une mise à jour.
    env_file:
      - /srv/<app>/.env
    environment:
      NODE_ENV: production
      LOG_LEVEL: info
      # Variables critiques : `:?` aborte le `compose up` si vide.
      BETTER_AUTH_SECRET: ${BETTER_AUTH_SECRET:?must be set}
      BETTER_AUTH_URL: ${BETTER_AUTH_URL:?must be set}

  caddy:
    image: caddy:2-alpine
    restart: always
    depends_on:
      - backend
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /srv/<app>/caddy/data:/data
      - /srv/<app>/caddy/config:/config
    environment:
      ACME_EMAIL: ${ACME_EMAIL:?must be set}
```

## Commandes

```bash
# Dev (depuis le repo local)
docker compose -f ops/docker-compose.yml -f ops/docker-compose.dev.yml up

# Prod (depuis le VPS)
docker compose -f ops/docker-compose.yml -f ops/docker-compose.prod.yml --env-file /srv/<app>/.env up -d --wait
```

`name: <app>` dans la base fixe le project name → `docker compose exec backend ...` sans `-f` marche même depuis n'importe quel répertoire dès qu'un container du projet tourne.

## `env_file:` vs `environment:` en prod

**Truc subtil** : sans `env_file:`, seules les vars listées explicitement dans `environment:` arrivent dans le container. Tu peux interpoler depuis `--env-file /srv/<app>/.env`, mais ça n'injecte pas la valeur dans le container — ça la rend juste disponible à Docker pour substituer `${VAR}` dans le yml.

Conséquence : si ta runbook dit "mets `RESEND_API_KEY=...` dans `/srv/<app>/.env`" mais que ton `docker-compose.prod.yml` ne liste pas `RESEND_API_KEY` dans `environment:`, ton backend ne la verra jamais.

**Solution propre** : `env_file: - /srv/<app>/.env` injecte tout. Tu gardes `environment:` pour les overrides explicites (ex. `NODE_ENV: production` pour forcer, ou `BETTER_AUTH_SECRET: ${BETTER_AUTH_SECRET:?must be set}` pour fail-closed sur var manquante).

C'est le bug qui aurait silencieusement cassé l'upload avatar R2 chez Listable jusqu'au refactor compose : les R2_* étaient dans `.env` mais pas mappés en `environment:`, donc absents du container.

## Validation

`docker compose ... config` rend la conf merged finale. Toujours run avant de commit un changement compose :

```bash
docker compose -f ops/docker-compose.yml -f ops/docker-compose.prod.yml \
  --env-file /tmp/fake-prod.env config 2>&1 | grep -A 5 'backend:'
```

À vérifier :
- En **prod merged** : `backend` a `image:` (pas de `build:`), aucune `volumes:` indésirable (les bind-mounts dev ne fuitent pas), `ports:` vide (Caddy fait le reverse proxy).
- En **dev merged** : `backend` a `build:` avec `target: development`, les bind-mounts hot-reload présents, port `127.0.0.1:3000` exposé.

## Pièges déjà rencontrés

1. **`volumes: []` ne reset rien** : compose append. Soit le `!reset []` (rustine), soit la base n'a pas de `volumes` du tout (clean — choisi ici).

2. **`command:` est replace, pas append** : si la base a `command: [a, b]` et l'overlay a `command: [c]`, le résultat final est `[c]`. Si tu veux les flags de base + des flags supplémentaires en prod, dupliquer.

3. **`name:` au top du yml** : sans ça, docker compose dérive le project name du nom du dossier parent. Si tu lances depuis `/srv/<app>` la prod aura un project name différent de la dev (lancée depuis `<repo>/ops`), et un `docker compose exec` ne trouvera pas les containers attendus.

4. **`build: !reset null` reste utile si la base a un build** : par exemple si la base a `build:` pour rendre les containers buildables localement même sans overlay. Le prod overlay fait alors `build: !reset null` + `image:` pour forcer le pull. Mais notre design ci-dessus évite ça en mettant `build:` uniquement dans `dev.yml`.

5. **Validation env vars avec `:?`** : préfère `${VAR:?must be set}` plutôt que `${VAR}` tout court. Le `:?` fail-closed à `compose up` si la var est vide, plutôt que de laisser le container booter avec une string vide et crasher à la 1ère requête.

---

**Dernière vérif** : 2026-05-13 (Listable v1).
**Projets concernés** : `listable`.
