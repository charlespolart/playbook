# App macOS hors App Store — Developer ID, notarisation, DMG

Le cas : une app qui a besoin de ce que le sandbox interdit (tap clavier
global, collage dans l'app frontale, Accessibilité) → **pas de Mac App Store**.
La seule voie propre : signature **Developer ID** + **notarisation** Apple, et
un `.dmg` que Gatekeeper ouvre sans détour. Vécu sur **Mumo** (dictée locale,
`com.overridedev.mumo`, équipe OVERRIDE) ; la mise à jour automatique est
dans `sparkle-auto-update.md`, le script de release dans
`release-runbook-pattern.md`.

## Configuration unique (une fois par machine)

1. **Certificat « Developer ID Application »** : Xcode → Settings → Accounts →
   équipe de l'**org** (jamais le compte perso — cf.
   `mobile/apple-team-id-and-bundle-id.md`) → Manage Certificates → `+` →
   Developer ID Application. Le rôle *Account Holder* est requis pour ce
   type de certificat. Vérif : `security find-identity -v -p codesigning |
   grep "Developer ID Application"` doit montrer `<ORG> (<TEAM_ID>)`.
2. **Notarisation** : un mot de passe d'application (appleid.apple.com →
   Connexion et sécurité) rangé dans un profil `notarytool`, jamais dans le
   repo :
   ```bash
   xcrun notarytool store-credentials <app>-notary \
     --apple-id <apple-id> --team-id <TEAM_ID> --password <mdp-application>
   ```
3. `node`/`npx` si le DMG est mis en page avec `appdmg` (ci-dessous).

Où ça vit dans le repo : `scripts/ExportOptions.plist` =
`method: developer-id`, `teamID`, `signingStyle: automatic`. Le Team ID et le
bundle id sont dans `docs/DISTRIBUTION.md`, pas devinés.

## Le pipeline (6 étapes, un seul script)

```
xcodegen generate                                   # projet régénéré (fichiers ajoutés/retirés)
xcodebuild -scheme <App> -configuration Release \
  -skipPackagePluginValidation -skipMacroValidation \
  -destination 'generic/platform=macOS' \
  -archivePath build/<App>.xcarchive DEVELOPMENT_TEAM=<TEAM_ID> archive
xcodebuild -exportArchive -archivePath … -exportPath build/export \
  -exportOptionsPlist scripts/ExportOptions.plist   # app signée, hardened runtime
codesign --verify --deep --strict --verbose=2 build/export/<App>.app
npx --yes appdmg build/dmg-spec.json build/<App>.dmg # fond + flèche → /Applications
xcrun notarytool submit build/<App>.dmg --keychain-profile <app>-notary --wait
xcrun stapler staple build/<App>.dmg && xcrun stapler validate build/<App>.dmg
```

Pourquoi ces choix :
- `-skipPackagePluginValidation` : mlx-swift embarque un plugin de build
  (`CudaBuild`, inactif sur macOS) que `xcodebuild` en ligne de commande
  refuse sans validation interactive.
- **`appdmg`** écrit directement le `.DS_Store` de la fenêtre d'install : aucune
  automatisation du Finder, donc aucun prompt de permission et ça tourne en
  arrière-plan. Repli `hdiutil create -srcfolder` si npx est indisponible.
- **Nom de volume ≠ nom de l'app** (`<App> Installer`) : le Finder met en cache
  le rendu de la fenêtre d'install **par nom de volume** ; un nom dédié évite
  qu'un vieux cache masque un nouveau fond.
- **Nettoyer `build/export` après coup** : un `.app` oublié dans `build/` est
  indexé par Spotlight (« 3 Mumo » dans la recherche, vécu). Garder l'archive :
  ses dSYM symbolisent les rapports de plantage.
- **arm64 seul** dès que MLX est dans le graphe (pas de x86_64) : `ARCHS`
  figé dans `project.yml`, et le README le dit (Intel non supporté).

Un build **Debug signé Developer ID** obtient la **même autorisation
Accessibilité** que la Release (TCC est indexé par l'exigence de code, pas
par le chemin) : la boucle de dev peut relancer l'app sans ré-accorder les
permissions à chaque fois.

## Pièges vécus

1. **`cannot execute tool 'metal' due to missing Metal Toolchain`** (Xcode
   27.2, 2026-11-04) : depuis Xcode 26, le compilateur Metal est un **composant
   à télécharger à part**. Le build Debug passait (noyaux déjà compilés en
   cache), seule l'**archive Release** cassait, parce qu'elle recompile les
   `.metal` de mlx-swift depuis zéro. Fix : `xcodebuild -downloadComponent
   MetalToolchain` (~1 Go), vérif `xcrun metal --version`. Toute mise à jour
   d'Xcode peut rejouer ce coup : le mettre dans le preflight de release.
2. **Modèle qui charge en 75–200 s au lieu de 2–3 s** après un disque plein :
   macOS a **purgé le cache des modèles compilés pour le Neural Engine**
   (démon `deleted`, urgence 3) → recompilation Core ML au prochain
   chargement. Ce n'est ni le build, ni la signature. Garder ≥ 10 Go libres
   sur la machine de dev, et prévenir les testeurs que le premier lancement
   compile.
3. **Notarisation `Invalid`** : `xcrun notarytool log <id> --keychain-profile
   <app>-notary` donne la raison exacte (hardened runtime manquant,
   binaire tiers non signé dans un package SPM, entitlement interdit).
4. **DMG monté pendant que l'app tourne** : `/Applications/<App>.app` est
   « en cours d'utilisation » et le remplacement échoue. Mumo détecte le
   montage d'un installeur plus récent, affiche une notice et **se ferme**
   ; au lancement depuis `/Applications` elle éjecte le volume d'install et
   propose de jeter le `.dmg`. À reproduire sur toute app distribuée en DMG.

## Première ouverture côté utilisateur

Trois permissions à accorder dans Réglages Système → Confidentialité : **Micro**,
**Accessibilité** (taper le texte à la place de l'utilisateur) et
**Surveillance de la saisie** (raccourci global). Les dire dans l'onboarding
et dans le README du dépôt de releases, sinon l'app « ne fait rien ».

## Symptômes à reconnaître

- `error: cannot execute tool 'metal'` sur l'archive seulement → piège 1.
- Premier chargement de modèle de plusieurs minutes, puis normal → piège 2.
- `spctl --assess` refuse le DMG → ticket non staplé ou notarisation `Invalid`.
- Plusieurs exemplaires de l'app dans Spotlight → export non nettoyé.

---

**Dernière vérif** : 2026-11-04 (Xcode 27.2, macOS 27.2 ; Metal Toolchain
vécu le jour même sur l'archive de Mumo 0.2.0/0.3.0).

Sources externes :
- [Apple — Notarizing macOS software before distribution](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)
- [Apple — Developer ID](https://developer.apple.com/developer-id/)
- [Apple — Xcode release notes (composant Metal Toolchain)](https://developer.apple.com/documentation/xcode-release-notes)
- [appdmg](https://github.com/LinusU/node-appdmg)
