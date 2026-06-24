# Android release signing — keystore + key.properties + Play App Signing

## Le piège qu'on mange en premier

Sur macOS, `/usr/bin/keytool` (et `apksigner`) sont des **stubs** qui
répondent *"Unable to locate a Java Runtime"* — il n'y a pas de JRE sur
le PATH. Pourtant Gradle builde l'APK sans problème : il utilise le JDK
**embarqué dans Android Studio** (le "JBR"), pas le PATH. Donc pour tout
ce qui touche aux clés/signatures, il faut pointer ce JDK :

```bash
JBR="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
KEYTOOL="$JBR/bin/keytool"
export JAVA_HOME="$JBR"   # nécessaire pour apksigner aussi
```

(`flutter doctor -v` affiche le chemin exact sous "Java binary at:".)

## Générer le keystore d'upload

Un keystore = **une app** (convention propre ; techniquement réutilisable
mais un par app évite les couplages). Mot de passe fort, généré, jamais
tapé à la main :

```bash
PW=$(openssl rand -base64 24 | tr '+/' 'Ab' | tr -d '=')   # shell-safe
mkdir -p "$HOME/keystores"
"$KEYTOOL" -genkeypair -v \
  -keystore "$HOME/keystores/<app>-upload.jks" \
  -alias upload \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass "$PW" -keypass "$PW" \
  -dname "CN=<App>, OU=Mobile, O=<org>, C=FR"
echo "MOT DE PASSE À SAUVEGARDER : $PW"
```

- `validity 10000` jours (~27 ans) : Play exige une clé valide au-delà de 2033.
- store password == key password : simplifie, accepté.
- Range le `.jks` **hors de tout repo git** (`~/keystores/`), jamais dans
  le projet — un `.jks` committé = clé compromise.

**Sauvegarde** : le fichier `.jks` **ET** le mot de passe dans un
gestionnaire de mots de passe (attache le fichier en pièce jointe, pas
juste le chemin). Sans les deux, impossible de publier une mise à jour.
Perte = ticket support Google pour reset l'upload key.

## Câblage Gradle (Kotlin DSL)

`android/key.properties` (gitignoré) :
```properties
storePassword=<pw>
keyPassword=<pw>
keyAlias=upload
storeFile=/Users/<you>/keystores/<app>-upload.jks
```

`android/app/build.gradle.kts` — lecture conditionnelle + **fallback
debug** (pour que `flutter run --release` et la CI sans secrets marchent
quand même) :
```kotlin
import java.util.Properties

val keystorePropertiesFile = rootProject.file("key.properties")
val keystoreProperties = Properties().apply {
    if (keystorePropertiesFile.exists()) {
        keystorePropertiesFile.inputStream().use { load(it) }
    }
}
val hasReleaseSigning = keystorePropertiesFile.exists()

android {
    signingConfigs {
        if (hasReleaseSigning) {
            create("release") {
                keyAlias = keystoreProperties["keyAlias"] as String
                keyPassword = keystoreProperties["keyPassword"] as String
                storeFile = (keystoreProperties["storeFile"] as String?)?.let { file(it) }
                storePassword = keystoreProperties["storePassword"] as String
            }
        }
    }
    buildTypes {
        release {
            signingConfig = if (hasReleaseSigning)
                signingConfigs.getByName("release")
            else
                signingConfigs.getByName("debug")   // build local OK, PAS uploadable
        }
    }
}
```

`.gitignore` (le template Flutter android le contient déjà — vérifie) :
```
key.properties
**/*.jks
**/*.keystore
```

Commit un `key.properties.example` documentant le format + la commande
keytool, pour le prochain dev / la prochaine machine.

## Vérifier que l'APK est bien signé par la bonne clé

`keytool -printcert -jarfile` ne lit QUE les signatures v1 (JAR) ; les
APK Flutter sont signés v2/v3 → sortie vide, c'est trompeur. Utiliser
`apksigner` (avec `JAVA_HOME` exporté) :

```bash
APKSIGNER=$(ls "$HOME/Library/Android/sdk/build-tools/"*/apksigner | sort -V | tail -1)
"$APKSIGNER" verify --print-certs build/app/outputs/flutter-apk/app-release.apk
# → "Signer #1 certificate SHA-256 digest: <hex>" doit matcher le keystore
```

## Récupérer les SHA-1 / SHA-256 (Firebase + App Links)

```bash
"$KEYTOOL" -list -v -keystore "$HOME/keystores/<app>-upload.jks" -storepass "$PW"
# → SHA1 + SHA256
```

Où ils servent :
- **Firebase Console** → app Android → ajouter **SHA-1 ET SHA-256** →
  re-télécharger `google-services.json`. Sans ça, `oauth_client: []`
  reste vide → Google Sign-In échoue (`DEVELOPER_ERROR`).
- **App Links** : le SHA-256 va dans le `assetlinks.json` servi par le
  backend (ex. env `ANDROID_SHA256_FINGERPRINTS`).

## ⚠️ Play App Signing : DEUX clés, pas une

Au premier upload d'AAB, Google génère **sa propre clé de signature
d'app** (App Signing key) et garde la tienne comme **clé d'upload**.
Conséquence : il y a deux SHA-256 différents.

- L'app distribuée aux users est signée par la clé **Google** → c'est
  CE SHA-256 que voient les users (Google Sign-In, App Links en prod).
- Ta clé d'upload ne sert qu'à signer ce que TU envoies à Play.

Règle officielle (Play docs) : *"the app signing certificate fingerprint
is used to register public key(s) with API providers"*. Donc **le SHA
qui compte pour la prod = celui de la App Signing key (Google)**, pas
l'upload key. En pratique : enregistre **les DEUX** SHA-256 (upload +
app signing) dans Firebase + `assetlinks.json` — l'upload couvre tes
builds locaux/internal-testing, l'app-signing couvre ce que les users
installent. Les deux fingerprints sont dans Play Console → Setup → App
signing (et la clé upload via keytool en local). Oublier l'app-signing
key = Google Sign-In / App Links cassent **en prod** alors que tout
marche en build local.

## Symptômes à reconnaître

- **`Unable to locate a Java Runtime`** sur keytool/apksigner : utilise le
  JBR d'Android Studio, pas `/usr/bin/keytool`.
- **`keytool -printcert -jarfile` ne sort rien** : APK signé v2/v3,
  utilise `apksigner verify --print-certs`.
- **Google Sign-In `DEVELOPER_ERROR` / échoue en prod mais pas en debug** :
  le SHA de la clé **App Signing** (Google) n'est pas dans Firebase. Il y
  en a deux à enregistrer.
- **App Links ouvrent le navigateur au lieu de l'app** : `assetlinks.json`
  renvoie 404 (fingerprint env vide) ou le mauvais SHA.
- **AAB rejeté par Play "signed with debug key"** : `key.properties`
  absent → le fallback debug a signé. Vérifie qu'il existe + pointe le bon
  `.jks`.

---

**Dernière vérif** : 2026-06-03 (macOS, Flutter 3.44, Android Studio JBR 21).

Sources externes consultées :
- [Flutter — Build and release an Android app (signing)](https://docs.flutter.dev/deployment/android#signing-the-app)
- [Play Console — App signing](https://support.google.com/googleplay/android-developer/answer/9842756)
- [apksigner](https://developer.android.com/tools/apksigner)
