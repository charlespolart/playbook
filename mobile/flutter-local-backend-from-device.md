# Flutter debug sur un vrai téléphone — joindre le backend local sans taper d'IP

## Le piège

En debug, l'app parle à `http://localhost:3001` (choix compile-time via
`kDebugMode`, cf. `flutter-mobile-build.md`). Sur le simulateur iOS c'est le
Mac, donc ça marche. Sur un **iPhone ou un Android physique**, `localhost`
c'est le téléphone : `Connection refused` sur tout, création de compte et
Sign-In Google compris. On croit à un bug d'auth, c'est juste le host.

La tentation : coller l'IP du Mac dans le code ou un `.env`. Elle change à
chaque Wi‑Fi (DHCP), on l'oublie dans un commit, et un jour elle shippe.

## La règle : deux mécanismes, un par OS, et la release intouchable

| Plateforme | Comment le téléphone atteint le Mac | Ce que l'app utilise |
|---|---|---|
| Android (USB) | `adb reverse tcp:3001 tcp:3001` : le `localhost:3001` **du téléphone** est tunnelé vers le Mac | `localhost`, inchangé |
| iPhone (même Wi‑Fi) | le **nom Bonjour** du Mac, `<LocalHostName>.local`, résolu par mDNS | ce nom, passé au build par `--dart-define=API_HOST=` |
| Simulateur iOS | rien à faire | `localhost` (ou le `.local`, les deux marchent) |

Pourquoi pas le nom Bonjour partout ? Pas de tunnel inverse USB sur iOS
(Apple ne l'offre pas, usbmuxd ne va que Mac → iPhone), et la résolution
`.local` est peu fiable dans les apps Android. Chacun son outil.

**Le `--dart-define` n'est lu que dans la branche debug.** La règle
« l'URL d'API ne passe jamais par dart-define » de `flutter-mobile-build.md`
tient toujours : une release ne peut pas hériter d'un host de dev.

## Le code

`lib/services/dev_host.dart` :

```dart
const String apiHostDefine = String.fromEnvironment('API_HOST', defaultValue: 'localhost');

/// Android garde localhost (adb reverse) ; iOS prend le nom passé par le Makefile.
String devApiHost({required bool android, String define = apiHostDefine}) => android ? 'localhost' : define;
```

Dans le service API et le service WebSocket, uniquement la branche debug :

```dart
static String get baseUrl =>
    kDebugMode ? 'http://${devApiHost(android: Platform.isAndroid)}:3001' : 'https://api.exemple.com';
// idem pour 'ws://…' / 'wss://…'
```

Trois tests suffisent : Android → `localhost` quel que soit le define ; iOS →
le define ; sans define → `localhost`.

## Le Makefile

```make
ADB      := $(HOME)/Library/Android/sdk/platform-tools/adb
API_PORT := 3001

run:            ## Debug sur appareil/simulateur ; `make run d=<device id>` pour choisir
	-@[ -x "$(ADB)" ] && for s in $$("$(ADB)" devices | awk 'NR>1 && $$2=="device"{print $$1}'); do "$(ADB)" -s $$s reverse tcp:$(API_PORT) tcp:$(API_PORT) >/dev/null && echo "adb reverse: $$s:localhost:$(API_PORT) → this Mac"; done; true
	cd apps/mobile && flutter run --dart-define=API_HOST=$$(scutil --get LocalHostName).local $(if $(d),-d $(d),)
```

- `adb reverse` est **par appareil et volatile** (perdu au débranchement, au
  redémarrage du serveur adb) : c'est pour ça qu'il est refait à chaque
  `make run`, pour tous les appareils en état `device`.
- `scutil --get LocalHostName` donne le nom Bonjour (pas `hostname`, qui
  peut renvoyer un nom DHCP du routeur). Vérifier qu'il est annoncé :
  `dns-sd -G v4 <nom>.local` doit lister l'IP du Mac.

## Côté iOS : deux clés dans `Info.plist`, sinon rien ne passe

```xml
<key>NSAppTransportSecurity</key>
<dict>
	<key>NSAllowsLocalNetworking</key>
	<true/>
</dict>
<key>NSLocalNetworkUsageDescription</key>
<string>Only a debug build uses the local network, to reach a development server on your own computer.</string>
```

- `NSAllowsLocalNetworking` : autorise le HTTP en clair vers les hôtes
  `.local` et les IP locales, et rien d'autre (la release, en HTTPS vers la
  prod, n'est pas concernée).
- `NSLocalNetworkUsageDescription` : depuis iOS 14, **résoudre un `.local`
  ou joindre une IP privée demande la permission « Réseau local »**. Sans
  cette clé, pas de prompt, et l'échec ressemble à une erreur DNS.

## Côté backend et Mac

- Le serveur doit écouter sur **toutes les interfaces** : `server.listen(port)`
  sans host en Node est bon ; `127.0.0.1` ne l'est pas.
- Pare-feu macOS : s'il est actif, accepter la demande « autoriser `node` à
  recevoir des connexions entrantes ».
- Le Mac et l'iPhone sur le **même Wi‑Fi**, pas en 4G, pas sur un réseau
  invité (isolation client), pas en partage de connexion.

## Symptômes à reconnaître

- **`Connection refused` sur tout, dès l'inscription, en `flutter run` sur
  un vrai téléphone** : `localhost` = le téléphone. Pas un bug d'auth.
- **iPhone : `SocketException: Failed host lookup: 'xxx.local' … errno = 8`**
  alors que `dns-sd -G v4 xxx.local` répond sur le Mac : permission
  « Réseau local » refusée ou jamais demandée (Réglages → Confidentialité et
  sécurité → Réseau local → l'app doit y être, activée ; sinon supprimer
  l'app et relancer pour revoir le prompt), ou pas le même réseau.
- **Android : ça marchait, plus maintenant** : `adb reverse --list` vide →
  le tunnel est tombé (câble, `adb kill-server`). Relancer `make run`.
- **`INSTALL_FAILED_UPDATE_INCOMPATIBLE … signatures do not match`** en
  passant d'une release locale à un `make run` (ou l'inverse) : clés
  différentes, Flutter désinstalle puis réinstalle. Les données locales de
  l'app sur l'appareil partent avec.
- **Sign-In Google qui se ferme sans message sur un build release local
  Android**, logcat `UNREGISTERED_ON_API_CONSOLE` / « not registered to use
  OAuth2.0, please confirm the package name and SHA-1 » : rien à voir avec
  le réseau, c'est la clé d'upload pas déclarée en client OAuth Android →
  `android-release-signing.md` (2 SHA). En debug (clé de debug déclarée)
  ça passe.

**Dernière vérif** : 2026-10-28 (Dian Dian : Retroid Pocket 4 Pro en Android
13 par `adb reverse`, iPhone par `Charless-Laptop.local` ; Flutter 3.47).

## Liens

- [adb — forward / reverse ports](https://developer.android.com/tools/adb#forwardports)
- [Apple TN3179 — Understanding local network privacy](https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy)
- [Apple — NSAllowsLocalNetworking](https://developer.apple.com/documentation/bundleresources/information-property-list/nsapptransportsecurity/nsallowslocalnetworking)
- [Flutter — `--dart-define` et `String.fromEnvironment`](https://docs.flutter.dev/deployment/flavors#using-dart-define)
