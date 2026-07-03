# Team ID Apple + bundle id : toujours DEMANDER, jamais deviner

## Le piège

Au setup iOS (ou au premier signing), c'est tentant de **deviner** le
`DEVELOPMENT_TEAM` et le `PRODUCT_BUNDLE_IDENTIFIER` — depuis le nom du dev
de la machine, ou en prenant la première équipe Apple trouvée. **Ne le fais
jamais.** Vu en vrai sur un projet : `DEVELOPMENT_TEAM = V54ULTRA7C` posé par
une session précédente (matchait **aucun** cert installé → signature device
cassée) et `PRODUCT_BUNDLE_IDENTIFIER = com.charlespolart.lifeAnswerMobile`
(nom du dev). Le bon était l'équipe **pro** (OVERRIDE, `MKW79AHB7A`) et
`com.overridedev.lifeAnswerNotebook`.

Conséquences d'un mauvais choix :
- **Team faux** → impossible de signer sur un device réel (pas de cert/profil),
  ou pire : l'app finit publiée **sous le mauvais compte Apple Developer**.
- **Bundle id faux** → App ID enregistré sous le mauvais reverse-domain ;
  douloureux à changer après publication (= nouvelle app sur le Store).

## La règle

À la mise en place iOS et avant tout build device/store, **demander
explicitement à Charles** :
1. **Quel Team ID** utiliser — il a plusieurs équipes (perso
   `mario.polart@hotmail.fr`, pro **OVERRIDE**, éventuellement d'autres). Le
   bon dépend du projet : **prod / app publiée = l'équipe de l'org**, jamais le
   compte perso.
2. **Quel bundle id / applicationId** — convention `com.<org>.<app>`
   (ex. `com.overridedev.lifeAnswerNotebook`), **pas** `com.<nom-du-dev>.<app>`.

Ne pas auto-détecter, ne pas réutiliser celui d'un autre projet par défaut.

## Vérifier un Team ID → quelle org

```bash
security find-identity -v -p codesigning
```
Liste les certs **installés** avec le Team ID entre parenthèses et le nom
d'org, ex. `Developer ID Application: OVERRIDE (MKW79AHB7A)` ou
`Apple Development: mario.polart@hotmail.fr (6QFQ6HKJHD)`.

⚠️ **Limite** : ça ne montre que les certs présents sur la machine. Une
appartenance d'équipe sans cert encore généré n'apparaît pas — donc si le team
voulu n'est pas dans la liste, **confirmer avec Charles** plutôt que conclure
qu'il est faux. Source autoritaire : Xcode → Settings → Accounts → `<org>` →
*Team ID*, ou [developer.apple.com → Membership](https://developer.apple.com/account).

## Où c'est posé

- **iOS** : `apps/<mobile>/ios/Runner.xcodeproj/project.pbxproj`
  - `DEVELOPMENT_TEAM` → **3 occurrences** (Debug / Release / Profile).
  - `PRODUCT_BUNDLE_IDENTIFIER` → 3× la cible `Runner` + 3× `RunnerTests`
    (le `.RunnerTests` doit rester un suffixe : `<bundle>.RunnerTests`).
  - `Info.plist` utilise `$(PRODUCT_BUNDLE_IDENTIFIER)` (variable) → rien à
    toucher là.
- **Android** : `apps/<mobile>/android/app/build.gradle.kts` → `applicationId`
  (= le bundle id côté Android ; le garder **cohérent** avec l'iOS).

Après changement du bundle id, `flutter clean` puis rebuild (les artefacts
portent l'ancien id).

## Signature device : ce qui doit être vrai

`flutter run` sur un iPhone réel en **automatic signing** (`CODE_SIGN_STYLE =
Automatic`) marche seulement si l'Apple ID connecté dans Xcode est **membre de
l'équipe** choisie avec un rôle qui autorise la création de certs/profils.
Sinon Xcode échoue à provisionner. Si le device build refuse de signer après
avoir mis le bon team, vérifier l'appartenance de Charles à l'org dans Xcode →
Accounts.

---

**Dernière vérif** : 2026-06-25 (Xcode 27, Flutter 3.44, automatic signing).

Sources externes :
- [Apple — Team ID / Membership details](https://developer.apple.com/help/account/manage-your-team/locate-your-team-id/)
- [Flutter — iOS deployment / signing](https://docs.flutter.dev/deployment/ios)
