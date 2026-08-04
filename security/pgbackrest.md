# pgBackRest — activation, pièges, et test de restore

Backups Postgres chiffrés hors-site avec PITR. Cette fiche couvre **ce qui
casse à l'activation** : une config « prête dans le repo » n'est pas une config
qui marche, et les deux projets qui l'ont activée (2026-08) ont eu leur premier
backup en échec, pour des raisons différentes.

Le choix pgBackRest vs `pg_dump + age + rclone` se fait dans
`data-protection.md` (section Backups). Ici on suppose le choix fait.

## Architecture : pas de sidecar

pgBackRest lit les fichiers de données **directement** : il lui faut PGDATA sur
son propre système de fichiers. Un sidecar aurait besoin soit d'SSH/TLS vers
postgres (`pg1-host=…`), soit d'un partage du volume de données. Une config qui
déclare `pg1-host=postgres` **sans configurer l'un des deux** échoue au premier
backup — c'est le défaut qu'on trouve dans la plupart des exemples copiés.

Le montage qui marche :

- le client pgbackrest **dans l'image postgres** (`FROM postgres:16-alpine` +
  `apk add pgbackrest`), pour qu'`archive_command` le trouve localement ;
- `pg1-host` **absent** → accès local, via `pg1-socket-path=/var/run/postgresql` ;
- la **planification par un cron hôte** qui fait `docker compose exec`.

Le cron hôte a un bénéfice de sécurité en prime : il évite de monter
`/var/run/docker.sock` dans un conteneur (= root sur l'hôte, cf.
`vps-hardening.md`).

## Les 6 pièges d'activation (tous vécus)

**1. Tout tourne en `--user postgres`** (uid 70 sur alpine, 999 sur debian).
`archive_command` est exécuté par postgres lui-même. Une commande pgbackrest
lancée en root — **c'est le défaut de `docker compose exec`** — crée
`/tmp/pgbackrest` en `root:root`, et l'archivage meurt derrière avec :

```
[041] unable to get info for path/file '/tmp/pgbackrest/<stanza>.stop': Permission denied
```

Deux parades, à appliquer **ensemble** :

```ini
# épingler lock/spool hors de /tmp, dans un volume possédé par postgres
lock-path=/var/lib/pgbackrest/lock
spool-path=/var/lib/pgbackrest/spool      # requis par archive-async
```
```bash
chown -R 70:70 /srv/<app>/pgbackrest/{lib,log}   # un bind mount écrase le chown de l'image
docker compose exec -T --user postgres postgres pgbackrest ...
```

**2. Le catch-22 du premier démarrage.** Si `archive_mode=on` est actif avant
que la stanza n'existe, `archive-push` sort en code 103 ; **postgres interprète
ça comme un backend planté et redémarre le cluster en boucle** (~11 s de cycle,
subi ~4 min en prod sur un projet). Or `stanza-create` réclame un primaire
stable. Sortie : désactiver l'archivage → `stanza-create` → réactiver. Mettre un
`--wait` dans le script d'init pour ne pas y revenir.

**3. Recréer le conteneur postgres casse le pool du backend.** Voir le principe
général dans `infrastructure/github-actions-deploy.md`, piège #13 : `/healthz`
continue de répondre 200 pendant que `/readyz` est faux. Toujours enchaîner par
un `up -d backend` et vérifier `/readyz`.

**4. L'overlay backup doit être dans TOUTES les commandes compose du deploy.**
Le piège le plus vicieux — silencieux et différé. Détail et correctif dans
`infrastructure/github-actions-deploy.md`, piège #12. Symptôme : l'archivage WAL
s'arrête, les backups « réussissent » encore, et on l'apprend au premier restore.

**5. Le credential objet doit être scopé au seul bucket de backups**, jamais
celui que porte le backend. Et il se vérifie **en négatif** (cf.
`data-protection.md`) : un token validé par un simple lecture/écriture sur sa
cible s'est révélé Admin sur tout le compte, voyant jusqu'aux buckets d'un autre
projet.

**6. La passphrase du dépôt ne va pas dans le `.env` commun**, que `env_file:`
injecte en entier dans le backend — cf. `data-protection.md`, section Secrets
ops. Elle vit dans `/srv/<app>/.env.pgbackrest` (root, 0600), chargé via un
second `--env-file`.

## Test de restore — un backup jamais restauré n'existe pas

Version **non destructive**, exécutable sur le VPS de prod lui-même : restaurer
dans un répertoire isolé, démarrer un postgres jetable dessus, comparer les
counts, tout détruire. Aucun contact avec le data dir de production.

> ⚠️ **Les 6 `PGBACKREST_*` sont nécessaires sur les DEUX conteneurs.** Le
> restore écrit un `restore_command = pgbackrest … archive-get` dans
> `postgresql.auto.conf`, et le cluster restauré doit rejouer du WAL pour
> atteindre la cohérence — WAL qui n'est pas dans le backup (sauf
> `--archive-copy=y`) et qu'il va donc chercher dans le dépôt. Sans les
> variables, l'entrypoint prend son chemin « pgbackrest not configured,
> skipping », aucun `/etc/pgbackrest/pgbackrest.conf` n'est rendu,
> `archive-get` échoue et la recovery reste bloquée sur
> `could not locate required checkpoint record`.

```bash
set -a; . /srv/<app>/.env.pgbackrest; set +a
RT=/srv/<app>/restore-test; rm -rf "$RT"; mkdir -p "$RT"; chown 70:70 "$RT"

# 1. Restaurer depuis le dépôt chiffré, dans un répertoire isolé.
docker run --rm --user postgres -v "$RT":/var/lib/postgresql/data/pgdata \
  -e PGBACKREST_S3_BUCKET -e PGBACKREST_S3_ENDPOINT -e PGBACKREST_S3_REGION \
  -e PGBACKREST_S3_KEY -e PGBACKREST_S3_KEY_SECRET -e PGBACKREST_REPO_CIPHER_PASS \
  <app>/postgres:16-pgbackrest \
  pgbackrest --stanza=<app> --type=immediate --target-action=promote restore

# 2. Démarrer un postgres jetable dessus — MÊMES variables (cf. encadré).
docker run -d --name restore-test --user postgres \
  -v "$RT":/var/lib/postgresql/data/pgdata -e PGDATA=/var/lib/postgresql/data/pgdata \
  -e POSTGRES_USER=<u> -e POSTGRES_DB=<db> -e POSTGRES_PASSWORD=throwaway \
  -e PGBACKREST_S3_BUCKET -e PGBACKREST_S3_ENDPOINT -e PGBACKREST_S3_REGION \
  -e PGBACKREST_S3_KEY -e PGBACKREST_S3_KEY_SECRET -e PGBACKREST_REPO_CIPHER_PASS \
  <app>/postgres:16-pgbackrest

# 3. Comparer les counts de 3-4 tables clés avec la prod.
docker exec restore-test psql -U <u> -d <db> -tAc "SELECT count(*) FROM users"
docker exec <pg-prod>   psql -U <u> -d <db> -tAc "SELECT count(*) FROM users"

# 4. Détruire : une copie restaurée est de la donnée utilisateur EN CLAIR.
docker rm -f restore-test && rm -rf "$RT"
```

`pg_is_in_recovery()` peut rester à `true` (hot standby lecture seule) : ça
n'invalide pas le test, les données sont là et interrogeables.

## Preuve de chiffrement — lire l'octet BRUT

Le critère est simple : un fichier du dépôt chiffré commence par `Salted__`
(en-tête OpenSSL) + octets aléatoires ; s'il commence par `[backrest]`, le
cipher n'est pas actif. **Mais la façon de le lire décide du résultat**, et deux
pièges donnent la conclusion inverse de la réalité :

1. **`pgbackrest repo-get` déchiffre de manière transparente.** Sans `--raw`, on
   lit `[backrest]` sur un dépôt parfaitement chiffré → fausse alerte.
2. Les warnings pgBackRest (`invalid option 's3-bucket'`, émis quand des
   variables `PGBACKREST_*` traînent dans l'environnement) partent sur
   **stdout** et polluent la capture — on finit par mesurer les warnings, pas le
   backup.

Deux méthodes fiables :

```bash
# A. Lire l'objet directement dans le stockage, sans passer par pgbackrest
#    (aucune couche de déchiffrement dans le chemin).
rclone cat "r2:<bucket>/pgbackrest/backup/<stanza>/backup.info" | head -c 32 | od -c

# B. Via pgbackrest, mais en BRUT et sans env parasite.
env -u PGBACKREST_S3_BUCKET -u PGBACKREST_S3_KEY -u PGBACKREST_S3_KEY_SECRET \
  pgbackrest --stanza=<stanza> repo-get --raw backup/<stanza>/backup.info | head -c 32 | od -c
```

Attendu : `S a l t e d _ _` suivi d'octets non imprimables. Un contrôle
complémentaire, insensible à la méthode : aucun marqueur INI ne doit apparaître.

```bash
... | head -c 4096 | grep -qa "backrest-version\|\[backrest\]" \
  && echo "!! EN CLAIR" || echo "OK: chiffré"
```

## Vérifier que ça tourne vraiment (24 h après)

```bash
docker exec -u postgres <pg> pgbackrest --stanza=<app> info   # status: ok + cipher: aes-256-cbc
psql -tAc "SELECT archived_count, failed_count FROM pg_stat_archiver"
journalctl -t <app>-backup --since '25 hours ago' | grep 'backup OK'
```

- `archived_count` doit **monter**. Un `failed_count` non nul mais **figé** =
  les échecs d'avant le correctif ; ce qui compte est qu'il n'augmente plus.
- La ligne `backup OK` doit tomber à chaque run : **silence = échec invisible**,
  un cron qui ne se déclenche plus ne prévient personne.
- `pgbackrest info` liste les backups : vérifier que le **full nocturne** est
  bien apparu tout seul, pas seulement les différentiels lancés à la main.

## Rétention, coût, dimensionnement

`repo1-retention-full=7` / `repo1-retention-diff=30` sur une base de ~30 Mo
tient dans quelques Mo compressés (zstd) — le coût objet est négligeable
(~0,05 $/mois pour 10 Go sur B2/R2). Le vrai risque de dimensionnement est le
**spool WAL** : plafonner `archive-push-queue-max` (1 GiB) pour qu'une panne du
stockage distant ne remplisse pas le disque du VPS. La continuité PITR est
cassée dans ce cas, mais la base reste debout — c'est le bon compromis.

## Liens

- pgBackRest — user guide : https://pgbackrest.org/user-guide.html
- Configuration reference : https://pgbackrest.org/configuration.html
- `data-protection.md` — choix de la stratégie, secrets ops, vérif en négatif
- `infrastructure/github-actions-deploy.md` — pièges #12 (overlay) et #13 (readiness)

---

**Dernière vérif** : 2026-08-04 (activé en prod sur 2 projets, restores testés
des deux côtés ; extraite de `data-protection.md` qui devenait une fiche
pgBackRest déguisée ; snippet de restore corrigé — il manquait les
`PGBACKREST_*` sur le conteneur de vérification, ce qui bloquait la recovery).
