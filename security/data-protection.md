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
- **`env_file:` injecte le fichier ENTIER dans le conteneur** — donc tout
  secret posé dans `/srv/<app>/.env` est lisible par le backend, c'est-à-dire
  par le process le plus exposé d'internet. C'est acceptable pour ce dont il a
  besoin ; ça ne l'est pas pour le reste. Cas typique : **la passphrase de
  chiffrement des backups**. La mettre dans le `.env` commun revient à donner
  la clé de déchiffrement des sauvegardes à celui qui se fera compromettre en
  premier — et à annuler l'essentiel du bénéfice du chiffrement. Parade :
  un **second fichier**, `/srv/<app>/.env.pgbackrest` (root, 0600), chargé
  uniquement par les scripts de backup et poussé au seul service concerné :
  ```bash
  docker compose --env-file /srv/<app>/.env \
                 --env-file /srv/<app>/.env.pgbackrest \
                 -f ... exec -T postgres pgbackrest ...
  ```
  (compose accepte plusieurs `--env-file`). Et l'ajouter au `.gitignore` :
  `.env` ne matche pas `.env.pgbackrest`.

### Vérifier une permission EN NÉGATIF (la leçon la plus rentable)

Une permission n'est vérifiée que si l'on a testé **ce qu'elle doit
interdire**. Tester le chemin nominal valide la *fonctionnalité*, pas la
*sécurité* — un token trop large passe ce test avec brio.

Vécu 2026-08 : un token de stockage objet créé « pour les backups » a été
validé par un simple lecture/écriture sur son bucket → OK. En le testant en
négatif, il s'est avéré **Admin sur tout le compte** : il listait les buckets
d'un **autre projet**, accédait au bucket public des avatars et pouvait créer
des buckets. Le test nominal n'aurait jamais rien montré.

Matrice à dérouler pour toute cred de stockage/API :

| Test | Attendu |
|---|---|
| Lire/écrire la ressource cible | ✅ autorisé |
| Lire une **autre** ressource du compte | ❌ 403 |
| **Créer** une ressource (bucket, clé) | ❌ 403 |
| Appeler l'**API d'administration** du provider | ❌ erreur d'auth |
| L'outil réel fonctionne avec les droits réduits | ✅ (sinon on a trop serré) |

La dernière ligne compte autant que les autres : réduire un scope sans
revalider l'outil derrière, c'est déplacer la panne au prochain incident.
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

pgBackRest est **activé en prod sur les deux projets depuis 2026-08** (R2, EU,
aes-256-cbc, PITR) → tous les pièges d'activation, le test de restore et les
vérifications sont dans **`security/pgbackrest.md`**. La voie `age + rclone`
ci-dessous reste valable pour un side-project dont les données se reconstruisent.

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

## Test de restore — un backup jamais restauré n'existe pas

Quelle que soit la voie choisie, la règle ne change pas : un backup non rejoué
n'est pas un backup. Trimestriel au minimum.

> Pour pgBackRest, la procédure non destructive (restore isolé + postgres
> jetable + comparaison des counts) et la preuve de chiffrement sont dans
> **`security/pgbackrest.md`** — avec leurs pièges, qui font la différence entre
> une vérification et une illusion de vérification.

**Voie simple (dump `age`)** :

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

**Dernière vérif** : 2026-08-04 (pgBackRest activé en prod sur les 2 projets,
restores testés des deux côtés ; pièges uid 70 / catch-22 stanza / overlay
absent du deploy / pool backend tous vécus ce jour-là ; ajout du piège
`env_file` et de la vérification négative des permissions).
