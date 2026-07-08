# Durcissement VPS — checklist jour 1 + le piège Docker/ufw

Objectif : réduire la surface d'attaque (ports, SSH) et le blast radius si
quelque chose fuit (moindre privilège partout). Un VPS neuf se fait scanner
dans les **minutes** qui suivent son allocation — la checklist se déroule
AVANT d'y mettre la moindre donnée.

## Checklist jour 1 (dans l'ordre)

```bash
apt update && apt full-upgrade -y

# 1. SSH par clé uniquement (garder une 2e session ouverte pendant le test !)
# 2. Firewall — allow SSH AVANT enable, sinon lock-out
ufw default deny incoming
ufw default allow outgoing
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 443/udp          # HTTP/3 (Caddy)
ufw enable

# 3. Anti-bruteforce SSH
apt install -y fail2ban    # jail sshd active par défaut sur Debian/Ubuntu

# 4. Patches de sécu automatiques
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades

# 5. Vérif DEPUIS LE LAPTOP (pas depuis le VPS) :
nmap -p- <ip-vps>          # attendu : 22, 80, 443 — rien d'autre
```

## SSH : clé uniquement

```bash
# /etc/ssh/sshd_config.d/00-hardening.conf
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password    # root OK mais par clé seulement
```

Deux pièges :

1. **L'ordre des drop-ins** : sshd prend la **première** valeur rencontrée
   pour chaque directive. Beaucoup d'images cloud déposent un
   `50-cloud-init.conf` avec `PasswordAuthentication yes` — un drop-in nommé
   `99-hardening.conf` passe **après** et perd. D'où le préfixe `00-`.
2. **Vérifier la config effective**, pas les fichiers :
   ```bash
   sshd -T | grep -Ei 'passwordauthentication|permitrootlogin|kbdinteractive'
   ```
   puis tester une nouvelle connexion **avant** de fermer la session courante.

Référence : recommandations OpenSSH de l'ANSSI —
https://cyber.gouv.fr/publications/usage-securise-dopenssh

## LE piège central : Docker ignore ufw

`ports:` dans un compose (ou `docker run -p`) écrit **directement dans
iptables** (chaîne DOCKER), traversée AVANT les règles ufw. Conséquence : un
`ports: "5432:5432"` est joignable depuis internet **même si `ufw status` dit
deny** — ufw ment sur ce que Docker publie.

Règles qui en découlent :

1. **En prod, seul Caddy a un bloc `ports:`** (80, 443/tcp, 443/udp). C'est
   un invariant à vérifier dans l'overlay prod à chaque nouvelle app.
2. Postgres/Redis/etc. : **pas de `ports:` du tout**. Le backend les joint par
   nom de service (`postgres:5432`) sur le réseau compose interne — publier
   n'apporte rien.
3. Besoin d'un accès admin ponctuel à la DB → binder localhost
   (`127.0.0.1:5432:5432` dans l'overlay **dev** uniquement) et passer par un
   tunnel : `ssh -L 5432:localhost:5432 root@<vps>`. Jamais `0.0.0.0`.
4. La vérité se teste **de l'extérieur** : `nmap -p- <ip>` depuis le laptop.
   Ni `ufw status` ni un scan local ne font foi.

Le fond du problème (chaîne DOCKER-USER si un jour il faut filtrer quand
même) : https://docs.docker.com/engine/network/packet-filtering-firewalls/

## fail2ban

Le défaut (jail `sshd`, 5 essais → ban 10 min) suffit. Pour durcir :

```ini
# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 1h
```

`fail2ban-client status sshd` pour voir les bans courants. Noter que les logs
**applicatifs** derrière Caddy voient l'IP du proxy — fail2ban ne protège ici
que SSH ; le bruteforce HTTP se gère au rate limiting applicatif (cf.
`app-security.md`).

## Mises à jour

- **OS** : unattended-upgrades (installé au jour 1) couvre les patches de sécu
  Debian/Ubuntu. Prévoir un reboot manuel quand `/var/run/reboot-required`
  existe (kernel).
- **Images Docker** : les CVE de l'image de base ne se corrigent qu'en
  **rebuildant** — un rythme de deploy normal suffit pour le backend. Pour les
  images tierces (postgres, caddy) : pin par version majeure
  (`postgres:16-alpine`, jamais `latest`) + un `docker compose pull && up -d`
  volontaire de temps en temps, pas d'agent auto (cf. anti-patterns de
  `github-actions-deploy.md`).

## Docker : limiter l'élévation de privilèges

Un container compromis (RCE dans le backend) ne doit pas donner le VPS :

- Process **non-root** dans l'image (`USER bun` — déjà la convention, cf.
  `github-actions-deploy.md` pour le chown des bind mounts).
- **Jamais** monter `/var/run/docker.sock` dans un container exposé : accès au
  socket = root sur l'hôte, c'est l'escalade la plus courante.
- Jamais `privileged: true` ; dans l'overlay prod, ajouter :
  ```yaml
  security_opt:
    - no-new-privileges:true
  ```
  (bloque les setuid/sudo DANS le container, gratuit, ne casse rien).

## Moindre privilège — récap des creds qui vivent sur/pour ce VPS

| Cred | Scope | Si elle fuite |
|---|---|---|
| Clé SSH deploy (secrets GitHub) | ce VPS uniquement, dédiée au CI | révoquer UNE clé dans authorized_keys, la clé perso survit |
| PAT GHCR sur le VPS | `read:packages` only | lecture des images privées, rien d'autre |
| `/srv/<app>/.env` | root 0600 | tout le projet → c'est LE fichier à protéger (cf. `data-protection.md`) |

Le principe : chaque cred répond à « qu'est-ce que ça coûte si elle fuite ? »
par le minimum possible, et se révoque indépendamment des autres.

## Vérifs périodiques (5 min/mois)

```bash
nmap -p- <ip-vps>                    # depuis le laptop — ports = 22/80/443 ?
docker compose ps                    # colonne PORTS : rien d'imprévu ?
fail2ban-client status sshd
lastb | head                         # tentatives de login échouées
df -h                                # disque plein = DoS auto-infligé (logs, images)
```

## Liens

- ANSSI (Open)SSH : https://cyber.gouv.fr/publications/usage-securise-dopenssh
- Docker & firewalls : https://docs.docker.com/engine/network/packet-filtering-firewalls/
- Docker security : https://docs.docker.com/engine/security/

---

**Dernière vérif** : 2026-07-08 (rédaction initiale).
