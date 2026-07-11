# Données : hash vs chiffrement vs clair, secrets, backups chiffrés

Le point clé pour choisir : **90 % des fuites passent par la couche
applicative** (injection, IDOR, cred volée) — et à ce niveau, la DB est déjà
déchiffrée pour servir l'app. Le chiffrement disque ne protège que contre le
vol physique (menace mineure en datacenter). Priorités réelles : contrôle
d'accès (cf. `app-security.md`), hash/chiffrement **ciblé** par type de
donnée, et backups chiffrés.

## En transit

- Web : TLS auto par Caddy (Let's Encrypt) + HSTS (cf. headers dans
  `app-security.md`). Rien d'autre à faire.
- Mobile : HTTPS only — ATS (iOS) le force par défaut, ne jamais ajouter
  d'exception `NSAllowsArbitraryLoads`.
- Intra-VPS : backend ↔ postgres sur le réseau Docker interne → TLS inutile.
  Le jour où la DB est **externe** (managée) : `sslmode=verify-full` dans la
  connection string, pas juste `require` (require chiffre mais ne vérifie pas
  l'identité du serveur).

## Au repos — quel traitement pour quelle donnée

| Donnée | Traitement | Pourquoi |
|---|---|---|
| Mots de passe | **Hash** argon2id — `Bun.password.hash()` | on n'a jamais besoin de la valeur, seulement de comparer ; irréversible par design |
| Tokens tiers stockés (OAuth refresh, API keys clients…) | **Chiffrement applicatif** AES-256-GCM, clé dans le `.env` du VPS | il faut pouvoir relire, mais un dump SQL (injection, backup volé) ne doit rien donner sans la clé |
| PII courante (email, nom…) | **Clair en DB** + contrôle d'accès + minimisation | la chiffrer casse index/recherche/unicité pour un gain quasi nul : la clé vit sur la même machine que la DB |
| Backups | **Toujours chiffrés** (age) avant de quitter le VPS | ils se baladent sur un stockage tiers |
| Disque VPS | Option du provider si dispo, sinon tant pis | ne protège que contre le vol physique du disque |

- À éviter : **pgcrypto** pour le chiffrement colonne — la clé transite dans
  chaque requête SQL et finit dans les logs postgres
  (`log_min_duration_statement`, crash reports).
- La meilleure protection d'une donnée reste de **ne pas la collecter** ou de
  la purger tôt (minimisation/rétention : cf. `legal/rgpd.md`).

## Chiffrement applicatif — snippet (crypto natif, zéro dep)

```ts
import { randomBytes, createCipheriv, createDecipheriv } from "node:crypto";

// ENCRYPTION_KEY : `openssl rand -base64 32`, vit dans le .env (jamais en DB)
const KEY = Buffer.from(process.env.ENCRYPTION_KEY!, "base64");

export function encrypt(plain: string): string {
  const iv = randomBytes(12);
  const c = createCipheriv("aes-256-gcm", KEY, iv);
  const enc = Buffer.concat([c.update(plain, "utf8"), c.final()]);
  return [iv, c.getAuthTag(), enc].map((b) => b.toString("base64")).join(".");
}

export function decrypt(blob: string): string {
  const [iv, tag, enc] = blob.split(".").map((s) => Buffer.from(s, "base64"));
  const d = createDecipheriv("aes-256-gcm", KEY, iv);
  d.setAuthTag(tag);
  return Buffer.concat([d.update(enc), d.final()]).toString("utf8");
}
```

GCM authentifie en plus de chiffrer (un blob modifié → `decrypt` throw, pas de
données corrompues silencieuses). IV aléatoire par valeur, stocké avec elle —
c'est normal, il n'est pas secret.

## Secrets ops

- `/srv/<app>/.env` : root, `0600`, jamais dans le repo. `$` littéral → `$$`
  (cf. `docker-compose-overlay.md`).
- **Jamais de secret dans l'image Docker** : un `ARG`/`ENV` de build reste
  visible dans `docker history`. Les secrets arrivent au runtime via
  `env_file` uniquement.
- CI : GitHub masque les secrets dans les logs, mais pas leurs
  transformations (base64, substring…) — ne jamais les `echo`, même
  "encodés".
- Un secret qui a fuité (commit, log, pastebin) se **rotate**, il ne se
  "supprime" pas — cf. `app-security.md` section secrets.

## Backups DB — chiffrés, hors VPS, testés

Les trois adjectifs sont obligatoires : un backup sur le VPS lui-même n'est
pas un backup (disque mort = tout perdu ; VPS compromis = backups compromis).

**Deux voies validées — choisir selon le besoin, pas mélanger :**

| | pg_dump + age + rclone (ci-dessous) | pgBackRest (voie Listable, `ops/pgbackrest/`) |
|---|---|---|
| Granularité | snapshot quotidien (on perd jusqu'à 24 h) | WAL archiving continu → PITR à la minute |
| Complexité | 10 lignes de bash, zéro service | service + config + stanza, mais restore outillé |
| Chiffrement | age, clé privée jamais sur le VPS | `repo1-cipher-type=aes-256-cbc` intégré |
| Quand | side-projects, données reconstructibles | prod avec users payants, données irremplaçables |

Listable a une config pgBackRest complète (S3/R2 + rétention + cipher) prête
dans `ops/pgbackrest/` — **non activée** tant que pas de 1er user payant
(décision 2026-04-29) ; en attendant c'est un `pg_dump` local NON conforme qui
tourne. À l'activation, c'est pgBackRest la cible, pas le script ci-dessous.

**Voie simple — chiffrement [age](https://github.com/FiloSottile/age)**
(moderne, simple, remplace gpg pour ce cas). Le point élégant : paire de clés
générée sur le **laptop** (`age-keygen`), seule la clé **publique** va sur le
VPS → un VPS compromis peut faire de nouveaux backups mais ne peut pas
**lire** les existants. La clé privée ne touche jamais le VPS.

```bash
#!/usr/bin/env bash
# /srv/<app>/ops/scripts/backup.sh — root, chmod 0700
set -euo pipefail
cd /srv/<app>
COMPOSE="docker compose --env-file .env -f ops/docker-compose.yml -f ops/docker-compose.prod.yml"

$COMPOSE exec -T postgres pg_dump -U <db-user> --no-owner <db-name> \
  | age -r <age-clé-publique> \
  | rclone rcat "r2:<bucket-backups>/pg-$(date +%F).sql.age"

rclone delete --min-age 30d "r2:<bucket-backups>/"   # rétention 30 j
echo "backup OK $(date -Is)"
```

```cron
# /etc/cron.d/<app>-backup
PATH=/usr/local/bin:/usr/bin:/bin
17 4 * * * root /srv/<app>/ops/scripts/backup.sh 2>&1 | logger -t <app>-backup
```

Pièges :

1. **cron ≠ shell interactif** : PATH minimal (d'où la ligne `PATH=` — age et
   rclone sont souvent dans /usr/local/bin), pas de tty (d'où `exec -T`).
2. **`set -euo pipefail` obligatoire** : sans pipefail, un `pg_dump` qui meurt
   en cours laisse age/rclone finir "avec succès" → backup tronqué qui a l'air
   valide.
3. **Silence = échec invisible** : vérifier dans `journalctl -t <app>-backup`
   que la ligne "backup OK" tombe chaque nuit ; un cron cassé ne prévient
   personne (même logique que la sentinelle du deploy).

**Test de restore (trimestriel)** — un backup jamais restauré n'existe pas :

```bash
rclone cat "r2:<bucket-backups>/pg-<date>.sql.age" | age -d -i ~/.age/key.txt > /tmp/dump.sql
docker run -d --name restore-test -e POSTGRES_PASSWORD=x postgres:16-alpine
docker exec -i restore-test psql -U postgres -c "CREATE DATABASE t" && \
  docker exec -i restore-test psql -U postgres -d t < /tmp/dump.sql
# → SELECT count(*) sur 2-3 tables, comparer avec la prod, puis rm + docker rm -f
```

Lien RGPD : la durée de rétention des backups (30 j ci-dessus) fait partie des
durées documentées du registre, et c'est elle qui répond à « et mes données
dans vos backups ? » lors d'une demande d'effacement (cf. `legal/rgpd.md`).

## Liens

- age : https://github.com/FiloSottile/age
- OWASP Password Storage Cheat Sheet :
  https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- OWASP Cryptographic Storage Cheat Sheet :
  https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html

---

**Dernière vérif** : 2026-07-12 (réconciliation avec la voie pgBackRest de
Listable — la fiche l'ignorait alors qu'une config complète existait).
