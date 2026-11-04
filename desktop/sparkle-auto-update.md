# Mise à jour automatique macOS — Sparkle 2, dépôt public séparé, feed raw

Pour une app hors App Store, la mise à jour c'est **Sparkle 2** : un flux RSS
(`appcast.xml`) signé **EdDSA**, et un `.app` zippé par version. Vécu sur
**Mumo** (spec `docs/superpowers/specs/2026-07-22-sparkle-auto-update-design.md`
dans le repo de l'app). Le build/notarisation est dans
`macos-distribution-hors-store.md`, le script dans `release-runbook-pattern.md`.

## Les trois décisions structurantes

1. **Update = `.app` zippé, le DMG reste pour le premier téléchargement.**
   Zipper avec `ditto -c -k --keepParent <App>.app <App>-<ver>.zip` : `ditto`
   préserve les attributs étendus de signature, `zip` non.
2. **Un dépôt GitHub public dédié aux binaires** (`<app>-releases`), la source
   restant privée. Raison : les assets d'une release d'un **repo privé ne se
   téléchargent pas sans token** → Sparkle ne pourrait jamais les récupérer.
   Le dépôt public n'a que `appcast.xml`, `CHANGELOG.md`, un README et les
   releases. Activer les *Issues* dessus : c'est le canal des testeurs.
3. **Feed servi par `raw.githubusercontent.com/<owner>/<app>-releases/<branche>/appcast.xml`**,
   **pas GitHub Pages**. Piège vécu : Pages sur un repo de projet applique
   **automatiquement le domaine personnalisé du compte** → l'appcast se
   retrouve sous le domaine perso de l'auteur. Pages désactivé, point.
   Vérifier la branche par défaut (`master` ou `main`) : elle est dans l'URL.

## Les clés — l'actif irremplaçable

`generate_keys` (livré dans le package SPM Sparkle, sous
`DerivedData/…/SourcePackages/artifacts`) crée la paire :
- la **publique** va dans `Info.plist` → `SUPublicEDKey` ;
- la **privée** vit dans le trousseau macOS (« Private key for signing Sparkle
  updates ») et **nulle part dans un repo**. Export de secours
  `generate_keys -x <fichier>` → pièce jointe dans le gestionnaire de mots de
  passe ; restauration sur une autre machine `generate_keys -f <fichier>`.

Les apps installées n'acceptent **que** les updates signées par cette clé.
La perdre = plus aucune mise à jour possible pour la base installée, sauf à
redistribuer un DMG à la main. Avant que la clé existe, le code ne démarre pas
Sparkle tant que `SUPublicEDKey` vaut un placeholder (zéro risque runtime).

## L'item d'appcast — ce qui doit être vrai

```xml
<item>
  <title>Version 0.3.0</title>
  <pubDate>Tue, 04 Nov 2026 15:00:00 +0000</pubDate>          <!-- RFC 822, EN ANGLAIS : LC_ALL=C date -->
  <sparkle:version>3</sparkle:version>                       <!-- = CFBundleVersion, ENTIER, doit AUGMENTER -->
  <sparkle:shortVersionString>0.3.0</sparkle:shortVersionString>
  <sparkle:minimumSystemVersion>14.0</sparkle:minimumSystemVersion>
  <description xml:lang="fr"><![CDATA[<h3>Nouveautés</h3><ul>…</ul>]]></description>
  <enclosure url="https://github.com/<owner>/<app>-releases/releases/download/v0.3.0/<App>-0.3.0.zip"
             sparkle:edSignature="…" length="…" type="application/octet-stream" />
</item>
```

- Sparkle compare **`sparkle:version`** (l'entier `CFBundleVersion`) à celui
  installé. S'il n'augmente pas, **personne ne voit la mise à jour** — c'est LE
  bug silencieux de ce système, d'où le garde-fou du script de release.
- `sparkle:edSignature` + `length` sortent littéralement de `sign_update
  <zip>` (clé lue dans le trousseau). Une signature **invalide fait rejeter
  l'item avant même la détection** : pour un test local, omettre l'attribut
  plutôt que d'en mettre un faux.
- Chaque `<description>` localisée porte un **`xml:lang`**, sinon Sparkle
  n'en choisit aucune.
- L'ordre des items n'importe pas : Sparkle retient la version la plus haute.
  Insérer avant `</channel>` avec Python, pas `awk` : l'awk de macOS refuse une
  chaîne `-v` contenant des sauts de ligne (vécu à la première publication).

## Côté app — mises à jour discrètes

Rappels discrets Sparkle 2 : `supportsGentleScheduledUpdateReminders = true`
et `shouldHandleShowingScheduledUpdate → false` côté user driver, puis l'app
signale elle-même (badge sur l'icône de barre de menus, item « Mettre à jour
vers X… », une notice par version ; « Ignorer cette version » retire tout).
`checkForUpdatesInBackground()` ~10 s après le lancement. Signal fiable =
`didFindValidUpdate` ; `standardUserDriverWillHandleShowingUpdate` n'est pas
toujours appelé. Un check en arrière-plan **ne fait rien** si une session de
mise à jour est déjà ouverte.

Les notes de version viennent d'un `CHANGELOG.md` unique, lu par **trois
consommateurs** : la fenêtre Sparkle (langue de l'utilisateur), la GitHub
Release (anglais) et la fenêtre « Nouveautés » de l'app au premier lancement
d'une version. Un parseur Python côté script et un parseur Swift miroir côté
app, même grammaire, testés tous les deux.

## Tester sans publier

```bash
defaults write <bundle-id> <FeedURLOverride> -string http://localhost:8000/appcast.xml
python3 -m http.server 8000      # dans un dossier contenant un appcast de test
# relancer l'app, attendre 10 s, lire le journal de diagnostic (« update: … »)
defaults delete <bundle-id> <FeedURLOverride> ; defaults delete <bundle-id> SUSkippedVersion
```
`http` en local déclenche un avertissement ATS mais le flux charge ; macOS peut
aussi afficher un pop-up « réseau local » au premier accès — le refuser ne
bloque pas localhost. Le vrai
test de bout en bout reste : installer le build N−1, publier N, vérifier que
l'app **propose et installe** N, et qu'une signature altérée est refusée.

## Symptômes à reconnaître

- L'app ne propose jamais la mise à jour → `CFBundleVersion` pas incrémenté.
- « Mise à jour refusée, signature invalide » → mauvaise clé privée au
  trousseau (restaurer `generate_keys -f`).
- Item présent dans l'appcast mais jamais détecté en test local →
  `sparkle:edSignature` bidon, ou `<description>` sans `xml:lang`.
- Appcast servi sous le domaine perso de l'auteur → GitHub Pages activé.
- `**gras**` affiché littéralement dans la fenêtre de mise à jour → la
  `<description>` est du **HTML**, pas du Markdown : le générateur d'appcast
  doit convertir (`**x**` → `<strong>x</strong>`) après l'échappement HTML ;
  GitHub, lui, rend le Markdown tel quel (vécu 0.3.0, corrigé à la main dans
  le flux puis dans le script).
- `sign_update introuvable` → package Sparkle non résolu (relancer
  `xcodegen generate` / un build).

---

**Dernière vérif** : 2026-11-04 (Sparkle 2.10 ; première publication réelle
de Mumo en cours — le test de bout en bout « N−1 propose N » fait ce jour).

Sources externes :
- [Sparkle — Documentation](https://sparkle-project.org/documentation/)
- [Sparkle — Publishing an update](https://sparkle-project.org/documentation/publishing/)
- [Sparkle — Gentle reminders](https://sparkle-project.org/documentation/gentle-reminders/)
- [GitHub — About custom domains and GitHub Pages](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages)
