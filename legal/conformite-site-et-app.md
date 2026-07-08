# Conformité site web & app mobile (hors RGPD)

Ce que la loi FR/UE et les stores exigent d'un site ou d'une app, au-delà des
données perso (pour ça : `rgpd.md`). L'essentiel tient en quelques pages
statiques + deux exigences stores à connaître AVANT de soumettre.

## Mentions légales (LCEN — obligatoires sur tout site édité à titre pro)

Une page statique `/mentions-legales` avec :

- **Éditeur** : nom/raison sociale, SIREN, adresse, email de contact
  (micro-entrepreneur : nom + SIREN suffisent, l'adresse perso peut être
  remplacée par une domiciliation) ;
- **Directeur de la publication** (= toi) ;
- **Hébergeur** : nom, adresse, téléphone (ceux du provider VPS — infos sur
  leur propre page mentions légales).

Réf : https://entreprendre.service-public.fr/vosdroits/F31228

## CGU / CGV

- **CGU** : pas légalement obligatoires, mais indispensables dès qu'il y a des
  comptes — c'est ce qui permet de suspendre un compte abusif, limiter la
  responsabilité, définir la résiliation. Sans CGU acceptées, pas de base
  contractuelle propre.
- **CGV** : **obligatoires** en e-commerce B2C. Les points chauds :
  - droit de **rétractation 14 j** ; exception pour le contenu numérique
    fourni immédiatement **si** le client renonce expressément (la case
    « j'accepte l'exécution immédiate et renonce à mon droit de
    rétractation » avant l'accès) ;
  - **médiateur de la consommation** : adhésion obligatoire pour tout
    e-commerçant B2C, à citer dans les CGV ;
  - achats in-app iOS/Android : le remboursement/rétractation passe par
    Apple/Google, pas par nous — les CGV doivent le dire.
- Acceptation : case à cocher explicite à l'inscription (« clickwrap »), avec
  la date d'acceptation stockée en DB.

## App Store (Apple) — les 4 exigences qui bloquent une review

1. **Suppression de compte in-app** obligatoire dès que l'app permet d'en
   créer un. Doit être trouvable dans l'app (pas « envoyez-nous un email »).
   https://developer.apple.com/support/offering-account-deletion-in-your-app/
2. **Privacy Nutrition Labels** : déclarer exactement ce que l'app (et ses
   SDK : RevenueCat, Firebase…) collecte. Chaque SDK documente quoi déclarer —
   chercher « privacy labels » dans leur doc. Mentir = retrait.
   https://developer.apple.com/app-store/app-privacy-details/
3. **ATT** (App Tracking Transparency) : uniquement si tracking **cross-app**
   (pubs, data brokers). Analytics internes → pas d'ATT. Ne pas afficher le
   prompt ATT « par précaution » : le déclencher sans tracker est aussi un
   motif de rejet.
4. Liens **Privacy Policy + Terms** accessibles dans l'app — en WebView
   interne, pas d'éjection Safari (cf. `mobile/flutter-inapp-legal-webview.md`).

Biens numériques = IAP obligatoire (cf. `third-party/revenuecat.md`).

## Google Play

- **Data Safety form** : équivalent des labels Apple, même exigence
  d'exactitude vis-à-vis des SDK embarqués.
  https://support.google.com/googleplay/android-developer/answer/10787469
- **Suppression de compte** : in-app **et** une URL web publique de demande de
  suppression (exigée dans la Play Console) — prévoir la page
  `/delete-account` côté site.

## Accessibilité (European Accessibility Act — depuis le 28 juin 2025)

- Concerne les services B2C : e-commerce, banque, transport, ebooks,
  télécoms. **Exemption micro-entreprise** : < 10 salariés ET < 2 M€ de CA →
  un solo dev est exempt aujourd'hui, mais l'exemption saute si la boîte
  grossit ; services existants d'avant 2025 : deadline 2030.
- Le minimum intelligent même exempt (gratuit au départ, coûteux en
  retrofit) : HTML sémantique (labels de form, boutons = `<button>`),
  contrastes suffisants, navigation clavier possible. Référentiel : WCAG
  (RGAA pour le secteur public FR).
- Réf : https://ec.europa.eu/social/main.jsp?catId=1202

## Emails

- **Marketing B2C** : opt-in préalable (case non pré-cochée, distincte des
  CGU), lien de désinscription fonctionnel dans **chaque** email, identité
  claire de l'expéditeur.
- **Transactionnels** (reset password, reçus, alertes de service) : pas de
  consentement nécessaire — mais pas de marketing déguisé dedans, sinon ils
  requalifient.

## Si le projet héberge du contenu d'utilisateurs (UGC)

DSA : prévoir un **mécanisme de signalement** de contenu illicite et un point
de contact. Les obligations lourdes ne visent que les grandes plateformes,
mais le signalement + retrait des contenus manifestement illicites s'applique
à tout hébergeur de contenu tiers.

## Si un jour : données de santé

Hébergement certifié **HDS** obligatoire en France — ça ne se bricole pas sur
un VPS classique. Le jour où un projet touche à la santé, c'est une décision
d'architecture à prendre AVANT la première ligne de code.

---

**Dernière vérif** : 2026-07-08 (rédaction initiale ; EAA + exemption
micro-entreprise vérifiés en ligne ce jour).
