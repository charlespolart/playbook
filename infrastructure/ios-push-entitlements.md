# iOS push entitlements — `aps-environment` Debug vs Release

## Le piège

Une fois l'app Flutter wirée avec `firebase_messaging` + une clé APNs Auth Key côté Firebase, le réflexe est de mettre `aps-environment=development` dans `Runner.entitlements` et de passer à autre chose. **C'est suffisant pour `flutter run` mais ça casse les push en TestFlight et App Store.**

Apple a deux serveurs APNs **physiquement séparés** :
- `development` (sandbox) — accessible avec un Apple Development certificate (Debug build)
- `production` — accessible avec un Apple Distribution certificate (TestFlight, Ad Hoc, App Store)

Le même device n'a **pas le même device token** dans les deux environnements. Si l'app demande à iOS un token sandbox mais que Firebase essaye de pousser vers production avec ce token, Apple répond `400 BadDeviceToken` et la notif disparaît silencieusement. La clé `.p8` côté serveur configurée en "Sandbox & Production" couvre les deux **côté backend** — mais l'app elle-même doit demander explicitement l'environnement attendu.

Symptôme typique : push fonctionnent en `flutter run`, ne fonctionnent plus en TestFlight, ou warning `ITMS-90078: Missing Push Notification Entitlement` au moment de l'upload App Store Connect.

## Le bon pattern

Deux fichiers entitlements distincts, swap par build configuration dans le `.pbxproj` :

```
apps/mobile/ios/Runner/
├── Runner.entitlements            # aps-environment = development
└── Runner.Release.entitlements    # aps-environment = production
```

Dans `Runner.xcodeproj/project.pbxproj`, chaque `XCBuildConfiguration` du target Runner pointe vers le bon fichier :

```
97C147061CF9000F007C117D /* Debug */ = {
  buildSettings = {
    CODE_SIGN_ENTITLEMENTS = Runner/Runner.entitlements;
    ...
  };
};
97C147071CF9000F007C117D /* Release */ = {
  buildSettings = {
    CODE_SIGN_ENTITLEMENTS = Runner/Runner.Release.entitlements;
    ...
  };
};
249021D4217E4FDB00AE95B9 /* Profile */ = {
  buildSettings = {
    CODE_SIGN_ENTITLEMENTS = Runner/Runner.Release.entitlements;
    ...
  };
};
```

**Profile est aussi en production** : c'est le mode `flutter build ipa --profile`, déjà signé avec un Distribution profile.

Et déclarer le nouveau fichier dans la section `PBXFileReference` + `PBXGroup` du `.pbxproj` pour que Xcode le voie dans le navigator :

```
F81A81372FAE1B6F0058192C /* Runner.Release.entitlements */ = {
  isa = PBXFileReference;
  lastKnownFileType = text.plist.entitlements;
  path = Runner.Release.entitlements;
  sourceTree = "<group>";
};
```

Et dans le `children` du groupe Runner :

```
F81A81362FAE1B6F0058192B /* Runner.entitlements */,
F81A81372FAE1B6F0058192C /* Runner.Release.entitlements */,
```

Une fois fait, **`flutter run` reste en development, `flutter build ipa` part en production, sans intervention manuelle dans Xcode au moment du build**.

## Contenu typique des deux fichiers

`Runner.entitlements` :

```xml
<plist version="1.0">
<dict>
  <key>com.apple.developer.applesignin</key>
  <array><string>Default</string></array>
  <key>com.apple.developer.associated-domains</key>
  <array><string>applinks:my-domain.app</string></array>
  <key>aps-environment</key>
  <string>development</string>
</dict>
</plist>
```

`Runner.Release.entitlements` : identique, sauf `<string>production</string>` au lieu de `development`. Penser à **dupliquer toutes les autres capabilities** (Sign in with Apple, Associated Domains, etc.). Si on en ajoute une dans le Debug file plus tard, ne pas oublier de la propager dans le Release.

## Côté Apple Developer portal

Vérifier que le **bundle ID** a la capability `Push Notifications` activée (*Identifiers* → ton App ID → cocher Push Notifications). Sans ça, le code signing en Release sera rejected par Xcode avec :

```
Provisioning profile "iOS Team Provisioning Profile: <bundle-id>"
doesn't include the aps-environment entitlement.
```

Le `.p8` APNs Auth Key elle-même est commune aux deux environnements (configurer "Sandbox & Production" à la création) — pas de splitting côté serveur, c'est uniquement côté entitlements client.

## Vérification au premier `flutter build ipa`

Le bug Flutter [#129075](https://github.com/flutter/flutter/issues/129075) (ouvert à ce jour) mentionne des cas où `flutter build ipa` n'inclut pas `aps-environment` dans l'IPA final, même quand le `CODE_SIGN_ENTITLEMENTS` est défini. La cause habituelle est que le path est configuré uniquement dans l'UI Signing & Capabilities de Xcode au lieu du `.pbxproj`. Avec le pattern ci-dessus (déclaration dans le `.pbxproj` directement), le bug ne se déclenche pas.

Pour valider quand même :

```bash
flutter build ipa --release
unzip -p build/ios/ipa/*.ipa "Payload/Runner.app/embedded.mobileprovision" \
  | security cms -D \
  | grep -A1 aps-environment
```

Doit retourner `<string>production</string>`. Si c'est `development` ou absent, l'IPA partira avec un mismatch et les push iOS seront silencieusement morts en TestFlight/App Store.

## Pourquoi pas un seul fichier avec une variable Xcode ?

On pourrait être tenté d'utiliser une `${BUILD_CONFIG}` substitution dans `Runner.entitlements`, du genre `<string>$(APS_ENV)</string>`. **Ça ne marche pas** : Xcode ne fait pas la substitution dans le fichier entitlements au build time, contrairement à `Info.plist`. Le code signing va injecter le string littéral `$(APS_ENV)` dans la signature, ce qui n'est pas une valeur valide et fera échouer l'enregistrement APNs.

Apple documente officiellement qu'il faut deux fichiers séparés : voir [APS Environment Entitlement (Apple Developer)](https://developer.apple.com/documentation/bundleresources/entitlements/aps-environment) — "Build Configuration-Specific Entitlements" est listé comme l'approche recommandée. "You cannot use the same value for both configurations".

## Le piège annexe — `UIBackgroundModes` aussi nécessaire

`aps-environment` permet à l'app de recevoir des push **en foreground**. Pour réveiller l'app en background sur une push silencieuse (`content-available: 1`), il faut aussi `UIBackgroundModes` dans `Info.plist` :

```xml
<key>UIBackgroundModes</key>
<array>
  <string>remote-notification</string>
</array>
```

Pas besoin de le splitter Debug/Release — c'est le même pour les deux. Mais l'oublier ne déclenche pas d'erreur de build : ça fait juste que les push avec `content-available` n'invoquent pas le `BackgroundMessageHandler` Flutter en background.

## Symptômes à reconnaître

- **Push iOS marchent en `flutter run`, plus rien en TestFlight** : un seul fichier entitlements avec `development`. Splitter et rebuild.
- **Upload App Store Connect rejeté avec `ITMS-90078`** : aps-environment manquant ou en `development` dans l'IPA. Vérifier le `.pbxproj` Release config.
- **`Provisioning profile doesn't include aps-environment`** au build Release : Push Notifications pas activé dans le portal Apple Developer pour ce bundle ID, OU mauvaise valeur dans le entitlement (genre `production` activé avec un Dev cert).
- **Push content-available reçues mais Flutter ne les traite pas en background** : `UIBackgroundModes` manquant dans Info.plist.

---

**Dernière vérif** : 2026-05-15 (Listable, vs. Apple Developer Documentation officielle).
**Projets concernés** : `listable` (`com.overridedev.listable`).

Sources externes consultées :
- [APS Environment Entitlement — Apple Developer Documentation](https://developer.apple.com/documentation/bundleresources/entitlements/aps-environment)
- [Flutter — Build IPA not correctly setting entitlements (#129075)](https://github.com/flutter/flutter/issues/129075)
- [ITMS-90078 Missing Push Notification Entitlement — Ernest Chiang](https://www.ernestchiang.com/en/posts/2020/flutter-itms-90078-missing-push-notification-entitlement/)
