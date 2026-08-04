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
4. La vérité se teste **de l'extérieur** — mais depuis un point de vue *propre*
   (cf. section suivante : un scan depuis le laptop ment). Ni `ufw status` ni un
   scan local ne font foi.

Le fond du problème (chaîne DOCKER-USER si un jour il faut filtrer quand
même) : https://docs.docker.com/engine/network/packet-filtering-firewalls/

## Le scan de ports depuis le laptop MENT (vécu, 2 projets, 2026-08)

Un `nmap` / scan TCP depuis une machine perso peut rapporter **tous les ports
ouverts** alors que le VPS n'en écoute que trois. Cause : une box, un VPN, un
DNS menteur ou un portail captif qui **répond SYN-ACK à tout** (l'un des deux
audits a vu le DNS résoudre vers `198.18.0.40`, une plage de test). Conclusion
naturelle : « catastrophe, Redis et Postgres sont exposés » — entièrement faux.

Deux façons de démasquer, à faire **avant** de crier au feu :

1. **La bannière.** Un port 22 réellement ouvert renvoie *toujours*
   `SSH-2.0-...` à la connexion. Si un scan annonce 15 ports ouverts et
   qu'aucun — pas même le 22 — ne renvoie de bannière, c'est le réseau local
   qui répond, pas le VPS. Idem en parlant le protocole : un vrai Redis répond
   `+PONG` à `PING`, un vrai MySQL envoie son handshake d'emblée.
   ```bash
   python3 - <<'PY'
   import socket
   for p in (22, 6379, 5432):
       s = socket.socket(); s.settimeout(4)
       try:
           s.connect(("<ip-vps>", p)); s.settimeout(2)
           print(p, "->", s.recv(40)[:40])
       except Exception as e:
           print(p, "-> refusé/filtré", type(e).__name__)
       finally: s.close()
   PY
   ```
2. **Un point de vue propre** : refaire le scan depuis une VM cloud jetable
   (Scaleway/Hetzner à l'heure). C'est la méthode de référence quand le doute
   persiste.

**Aucune des deux vues ne suffit seule**, et elles ne répondent pas à la même
question :

| Outil | Fait autorité pour… | Angle mort |
|---|---|---|
| `ss -tlnup` **sur le VPS** | **infirmer** — aucun process en écoute ⇒ rien à joindre, quoi qu'en dise le scan | un listener sur `0.0.0.0` ne dit pas si le pare-feu laisse passer |
| Scan externe, **point de vue propre** | **confirmer** une exposition réelle | ment si le réseau intermédiaire répond à la place du VPS |

C'est exactement pour ça que la section précédente existe : Docker publie en
contournant ufw, donc « ufw dit deny » n'infirme rien et « ss dit que ça
écoute » ne confirme pas l'exposition. Croiser les deux est la seule lecture
fiable.

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

> **Piège : `apt install fail2ban` ne suffit pas sur une image journald-only**
> (vécu sur les **deux** projets, 2026-08). La jail `sshd` par défaut lit
> `/var/log/auth.log`, qui n'existe pas quand la distro ne fait plus tourner
> rsyslog. Le service part en `failed` avec
> `Failed during configuration: Have not found any log file for sshd jail` — et
> comme personne ne relit `systemctl status` après un `apt install`, on croit
> être protégé alors que rien ne tourne. Fix : `backend = systemd` dans la
> section `[DEFAULT]` de `jail.local` (alternative : réinstaller rsyslog).
> **Toujours vérifier après installation** :
> ```bash
> systemctl is-active fail2ban        # doit dire "active", pas "failed"
> fail2ban-client status sshd         # doit afficher la jail, pas une erreur
> ```
> Sur un VPS déjà scanné, les premiers bans tombent dans les secondes qui
> suivent — c'est le signe que ça marche (l'un des deux boxes en avait 8 d'un
> coup, avec ~295 000 tentatives échouées au compteur `lastb`).

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
| `/srv/<app>/.env.pgbackrest` | root 0600, chargé par les seuls scripts de backup | les backups → séparé du `.env` **exprès** (cf. `data-protection.md`, piège `env_file`) |

Le principe : chaque cred répond à « qu'est-ce que ça coûte si elle fuite ? »
par le minimum possible, et se révoque indépendamment des autres.

### La clé de deploy n'est dédiée que si on l'a vérifiée (2 projets sur 2, 2026-08)

La ligne « clé SSH dédiée au CI » du tableau décrit une **intention**. Dans les
faits, les deux VPS audités avaient pour seule entrée dans
`/root/.ssh/authorized_keys` la **clé personnelle du laptop** — que le CI
utilisait aussi, via le secret `VPS_SSH_KEY`. Toute la propriété « révoquer UNE
clé, la perso survit » était donc fausse : révoquer l'accès du CI aurait coupé
l'accès humain, et inversement. Le scénario que la séparation est censée
couvrir (un runner CI compromis, un secret GitHub qui fuite) n'était pas couvert.

**Le contrôle tient en une ligne** — autant de clés que de consommateurs :

```bash
awk '{print $1, $NF}' /root/.ssh/authorized_keys   # type + commentaire de chaque clé
```

Une seule ligne alors qu'un CI déploie = le défaut est là.

**La correction se fait sans fenêtre de coupure**, en ajoutant avant de retirer :

```bash
# 1. sur le laptop — une clé DÉDIÉE, jamais réutilisée ailleurs
ssh-keygen -t ed25519 -f ~/.ssh/<app>_deploy -N '' -C 'github-actions-deploy@<app>'

# 2. l'autoriser À CÔTÉ de la perso (ne rien retirer à ce stade)
ssh-copy-id -i ~/.ssh/<app>_deploy.pub root@<vps>

# 3. vérifier qu'elle marche AVANT de basculer le CI
ssh -i ~/.ssh/<app>_deploy -o IdentitiesOnly=yes root@<vps> 'echo ok'

# 4. basculer le secret, puis valider par un vrai deploy
gh secret set VPS_SSH_KEY < ~/.ssh/<app>_deploy
```

La privée ne vit ensuite que dans le secret GitHub (write-only) : elle est
jetable et se régénère en rejouant ces 4 étapes. Ne retirer une clé
d'`authorized_keys` qu'après un deploy réussi avec la nouvelle.

## Vérifs périodiques (5 min/mois)

```bash
ss -tlnup                            # SUR LE VPS — l'autorité : qui écoute vraiment ?
nmap -p- <ip-vps>                    # de l'extérieur — croiser avec ss (cf. scan menteur)
docker compose ps                    # colonne PORTS : rien d'imprévu ?
sshd -T | grep -Ei 'passwordauth|permitroot'   # la config EFFECTIVE, pas les fichiers
systemctl is-active fail2ban && fail2ban-client status sshd
lastb | wc -l                        # tentatives échouées : l'ordre de grandeur suffit
df -h                                # disque plein = DoS auto-infligé (logs, images, WAL)
```

## Ce que deux audits réels ont trouvé (2026-08)

Deux VPS montés depuis cette fiche, audités a posteriori. Le durcissement SSH
**n'avait été appliqué sur aucun des deux** — la fiche existait, l'étape avait
sauté. Constat identique des deux côtés :

| Trouvé | Impact |
|---|---|
| `PasswordAuthentication yes` + `PermitRootLogin yes`, root avec mot de passe | root brute-forçable depuis internet |
| ~209 000 et ~295 000 échecs de login dans `lastb` | attaque **en cours**, pas théorique |
| fail2ban absent | rien ne ralentit l'attaque |
| Clé perso servant aussi de clé CI (section précédente) | la révocation séparée, seule raison d'être de la clé de deploy, n'existait pas |

Enseignement : la checklist « jour 1 » n'est pas auto-appliquante. La vérifier
**sur un VPS existant** coûte 30 secondes (`sshd -T`, `systemctl is-active
fail2ban`, `lastb | wc -l`) et c'est le meilleur retour sur temps investi de
toute cette fiche.

## Liens

- ANSSI (Open)SSH : https://cyber.gouv.fr/publications/usage-securise-dopenssh
- Docker & firewalls : https://docs.docker.com/engine/network/packet-filtering-firewalls/
- Docker security : https://docs.docker.com/engine/security/

---

**Dernière vérif** : 2026-08-04 (audits croisés de 2 VPS en prod : SSH non
durci sur les deux, fail2ban cassé par l'absence de `/var/log/auth.log`, et
scans de ports depuis le laptop entièrement faussés par le réseau local).
