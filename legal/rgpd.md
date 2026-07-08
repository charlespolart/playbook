# RGPD — le minimum vital, projet par projet

**Donnée personnelle** = tout ce qui se rattache à une personne identifiable :
email, nom, mais aussi IP (donc les logs serveur), push token, device id,
`app_user_id` RevenueCat. Autrement dit : **tout projet avec des comptes est
concerné**, il n'y a pas de cas « trop petit pour le RGPD ».

Deux réflexes règlent 80 % du sujet :

1. **Minimisation** — on ne collecte pas « au cas où ». Chaque champ du schéma
   DB qui contient de la donnée perso doit avoir une justification.
2. **Rétention** — chaque donnée a une durée de vie documentée ET un mécanisme
   de purge qui tourne (un cron > une politique écrite jamais appliquée).

## Bases légales — laquelle pour quoi

| Traitement typique | Base légale |
|---|---|
| Compte, fourniture du service, support | Contrat |
| Factures, comptabilité | Obligation légale |
| Logs sécurité, anti-abus, rate limiting | Intérêt légitime |
| Newsletter, marketing, analytics non exemptés, pubs | **Consentement** |

Piège : le consentement est la **pire** base quand une autre s'applique
(révocable à tout moment, à prouver). Ne jamais demander le consentement pour
ce qui relève du contrat — ça crée l'obligation de fonctionner sans.

## Registre des traitements (art. 30 — obligatoire, même solo)

Un `registre.md` par projet suffit. Une ligne par traitement :

| Traitement | Finalité | Base légale | Données | Durée | Sous-traitants |
|---|---|---|---|---|---|
| Comptes utilisateurs | fourniture du service | contrat | email, hash mdp | vie du compte + 30 j (backups) | \<hébergeur\> (UE) |
| Abonnements | facturation | contrat | app_user_id, statut abo | vie du compte | RevenueCat (US, DPF), Apple/Google |
| Emails transactionnels | notifications de service | contrat | email | vie du compte | Resend (US, DPF) |
| Logs serveur | sécurité | intérêt légitime | IP, user agent | 6 mois | \<hébergeur\> |

Modèle CNIL : https://www.cnil.fr/fr/RGPD-le-registre-des-activites-de-traitement

## Droits des personnes (réponse sous 1 mois)

Accès, rectification, effacement, portabilité, opposition. En pratique, chaque
projet doit avoir **deux capacités** dès la conception :

1. **Exporter** les données d'un user (un JSON des rows le concernant suffit).
2. **Supprimer un compte proprement**, c'est-à-dire en cascade :
   - DB : `ON DELETE CASCADE` sur les FK **dès le schéma initial** — le
     retrofit sur une DB pleine est pénible ;
   - fichiers (R2 : avatars, uploads) ;
   - sous-traitants (RevenueCat a une API delete customer ; Firebase : delete
     user) ;
   - backups : ils **expirent naturellement** (30 j — cf.
     `security/data-protection.md`) ; documenter cette durée est la réponse
     acceptée à « et vos backups ? » — pas besoin de réécrire les archives.

Apple **exige de toute façon** la suppression de compte in-app (cf.
`conformite-site-et-app.md`) → autant la construire au début, elle sert les
deux obligations.

## Sous-traitants & transferts hors UE

- Chaque service qui voit de la donnée perso (hébergeur, R2, Resend,
  RevenueCat, Firebase…) est un **sous-traitant** → il faut un DPA. En SaaS
  c'est une page « Data Processing Agreement » à accepter ; garder le lien
  dans le registre.
- Vendor US → vérifier certification **Data Privacy Framework** ou clauses SCC
  dans le DPA. À coût égal, préférer les régions **UE** (VPS, bucket R2 en
  `eu`, région Resend/SES UE si dispo).
- La privacy policy liste les sous-traitants — la tenir à jour quand on en
  ajoute un.

## Durées de rétention — valeurs raisonnables

| Donnée | Durée |
|---|---|
| Compte + données associées | vie du compte, purge à la suppression |
| Backups | 30 j |
| Logs serveur (IP) | 6–12 mois max |
| Factures | 10 ans (obligation comptable — base légale distincte) |
| Prospects/leads sans réponse | 3 ans après le dernier contact |

## Cookies & traceurs (règles CNIL — web **et** SDK mobiles)

- **Pas de bannière nécessaire** si uniquement des traceurs « strictement
  nécessaires » : cookie de session/auth, panier, préférences (thème,
  langue), load balancing. Un SaaS classique sans analytics tiers n'a
  **pas besoin de bannière** — ne pas en mettre une par cargo cult.
- **Mesure d'audience sans consentement** : possible seulement si l'outil
  entre dans l'exemption CNIL (self-host type Matomo **configuré** pour ça,
  stats anonymes, pas de recoupement). GA4 n'y entre **pas** → GA4 = bannière
  de consentement obligatoire. Le plus simple pour nous : un outil exempté ou
  pas d'analytics du tout.
- **Tout le reste** (pubs, pixels sociaux, analytics non exemptés) :
  consentement **préalable** (rien ne se dépose avant le choix), refuser
  aussi simple qu'accepter, révocable.
- **Mobile : mêmes règles** — un SDK analytics/ads est un traceur (la CNIL a
  des recommandations dédiées aux apps mobiles depuis 2024). S'ajoute ATT
  côté iOS si tracking cross-app (cf. `conformite-site-et-app.md`).

Réfs : https://www.cnil.fr/fr/cookies-et-autres-traceurs et
https://www.cnil.fr/fr/cookies-solutions-pour-les-outils-de-mesure-daudience

## Violation de données

- Notification **CNIL sous 72 h** dès qu'il y a un risque pour les personnes
  (formulaire : https://www.cnil.fr/fr/notifier-une-violation-de-donnees-personnelles),
  + notification des personnes elles-mêmes si le risque est élevé.
- Tenir un **registre des violations**, y compris les incidents non notifiés
  (avec la raison de la non-notification).
- Réflexe en cas de fuite : rotation des secrets → évaluer ce qui a fuité →
  documenter (timeline, périmètre) → notifier si besoin. La doc AVANT le fix
  cosmétique : les 72 h partent de la découverte.

## Ce qu'on n'a (probablement) PAS à faire

- **DPO** : pas obligatoire pour un solo dev sans données sensibles à grande
  échelle.
- **AIPD** (analyse d'impact) : seulement pour les traitements à risque élevé
  (santé, biométrie, surveillance, mineurs à grande échelle…).
- **Mineurs** : consentement parental sous 15 ans en France — si un projet
  cible ce public, c'est un terrain à part (AIPD probable), ne pas y aller
  par accident.

## Liens

- Guide RGPD du développeur (CNIL — pensé pour nous, checklist par étape du
  cycle de dev) : https://lincnil.github.io/Guide-RGPD-du-developpeur/
- CNIL, l'essentiel pour les TPE : https://www.cnil.fr/fr/rgpd-par-ou-commencer

---

**Dernière vérif** : 2026-07-08 (rédaction initiale ; recommandation cookies
CNIL consolidée janv. 2026 et exemption mesure d'audience vérifiées en ligne
ce jour).
