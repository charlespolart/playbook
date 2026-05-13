# Flutter mobile build — Makefile + Fastlane + dart-defines

## Le piège qu'on vient de manger

Listable v1 avait `make ipa` qui injectait les secrets mobiles via `--dart-define`, mais `bundle exec fastlane ios beta` (le path réel pour TestFlight) builduait **sans aucun `--dart-define`**. Conséquence : un IPA shippé sur TestFlight avec `REVENUECAT_IOS_KEY` vide → paywall silencieusement dead, zéro souscription possible. Pas d'erreur de build, juste un binaire dégradé. Trouvé en audit, pas en prod, mais à 24h près.

## Le pattern propre

```
apps/mobile/.env.example       # liste toutes les vars (tracked)
apps/mobile/.env               # valeurs réelles (gitignored)
apps/mobile/scripts/
  dart-defines.sh              # UNE source de parsing
Makefile                       # call .sh via xargs
fastlane/Fastfile              # call .sh via Shellwords / re-parse en Ruby
```

**Une seule source de parsing du `.env`** réutilisée par tous les callers de build. Sinon Makefile et Fastfile divergent — typiquement le Makefile gagne des features et Fastlane reste cassé silencieusement (notre cas exact chez Listable v1).

## Mobile secrets vs backend secrets

À ne **jamais confondre** :

| | `apps/mobile/.env` | `apps/backend/.env` |
|---|---|---|
| Quoi | Secrets compile-time **baked dans l'IPA/AAB** | Secrets runtime lus par le serveur |
| Exemples | `REVENUECAT_IOS_KEY`, `GOOGLE_SERVER_CLIENT_ID` (Android), feature flags client | `BETTER_AUTH_SECRET`, `REVENUECAT_WEBHOOK_SECRET`, `R2_*` |
| Injection | `--dart-define=KEY=VALUE` au build | `env_file:` Docker Compose ou export shell |
| Présence au runtime | Dans le binaire | Dans `process.env` du backend |
| Rotation | Nécessite un rebuild + republication | Modif `.env` + restart container |

**Le webhook RC secret n'a rien à voir avec l'IPA.** C'est un secret backend. Cette confusion m'a fait perdre 20 minutes à debugger un test qui n'avait rien à voir avec le build mobile.

## Le script partagé `dart-defines.sh`

```bash
#!/usr/bin/env bash
# Émet un `--dart-define=KEY=VALUE` shell-quoté par ligne.
set -euo pipefail
env_file="${1:-apps/mobile/.env}"
[[ -f "$env_file" ]] || exit 0

while IFS= read -r raw || [[ -n "$raw" ]]; do
  line="${raw%$'\r'}"
  line="${line#"${line%%[![:space:]]*}"}"
  line="${line%"${line##*[![:space:]]}"}"
  [[ -z "$line" || "$line" == \#* ]] && continue
  key="${line%%=*}"
  value="${line#*=}"
  [[ "$key" == "$line" ]] && continue
  [[ "$key" =~ ^[A-Z][A-Z0-9_]+$ ]] || continue
  [[ -z "$value" ]] && continue
  case "$value" in
    \"*\") value="${value#\"}"; value="${value%\"}" ;;
    \'*\') value="${value#\'}"; value="${value%\'}" ;;
  esac
  printf '%q\n' "--dart-define=${key}=${value}"
done < "$env_file"
```

Points-clés :
- `printf %q` shell-escape **tout le token** (pas juste la valeur) → survit à `xargs` et à `Shellwords.split` côté Ruby
- Skip commentaires, lignes vides, keys non-`SHOUTING_SNAKE`, valeurs vides
- Strip les guillemets matchés (convention dotenv `KEY="value"`)
- Pas d'usage de substring négatif (macOS bash 3.2 ne supporte pas `${s:1:-1}`)

## Makefile target

```make
MOBILE_ENV := apps/mobile/.env
DART_DEFINES := $(shell ./apps/mobile/scripts/dart-defines.sh $(MOBILE_ENV) | xargs)

ipa:
	cd apps/mobile && flutter build ipa --release $(DART_DEFINES)
```

`xargs` collapse les newlines du script en une seule ligne d'args. Make passe ça verbatim à `flutter build`.

## Fastfile (Ruby)

Deux helpers à coder une fois, réutilisés par tous les lanes :

```ruby
require "shellwords"

MOBILE_ENV_FILE = "#{FLUTTER_DIR}/.env"

def mobile_env
  @mobile_env ||= begin
    if File.exist?(MOBILE_ENV_FILE)
      File.readlines(MOBILE_ENV_FILE).each_with_object({}) do |raw, hash|
        line = raw.strip
        next if line.empty? || line.start_with?("#")
        key, _, value = line.partition("=")
        next if value.empty?
        next unless key =~ /^[A-Z][A-Z0-9_]+$/
        if (value.start_with?('"') && value.end_with?('"')) ||
           (value.start_with?("'") && value.end_with?("'"))
          value = value[1..-2]
        end
        hash[key] = value
      end
    else
      {}
    end
  end
end

def dart_defines_for_mobile
  mobile_env.map { |k, v| Shellwords.escape("--dart-define=#{k}=#{v}") }.join(" ")
end

def require_mobile_env(*keys)
  missing = keys.reject { |k| !mobile_env[k].to_s.empty? }
  return if missing.empty?
  UI.user_error!(
    "Missing required dart-define keys in #{MOBILE_ENV_FILE}: #{missing.join(', ')}.\n" \
    "Copy apps/mobile/.env.example to apps/mobile/.env and fill them in."
  )
end

def flutter_build_ios
  defines = dart_defines_for_mobile
  sh "cd #{FLUTTER_DIR} && flutter build ipa --release --export-method=app-store #{defines}"
end
```

Puis dans chaque lane ship :
```ruby
lane :beta do
  require_env("APP_STORE_CONNECT_API_KEY_ID", ...)   # cred fastlane
  require_mobile_env("REVENUECAT_IOS_KEY")            # cred mobile baked
  certificates
  flutter_build_ios
  # ... upload
end
```

**Yes, c'est du parsing dupliqué** entre bash et Ruby (8 lignes chacun). Trade-off : pas de coupling cross-langage via parse-the-other's-output. Les deux parsers sont triviaux à auditer côte-à-côte. Si un jour la logique grossit, factoriser via une CLI plus puissante (genre `dotenv-cli`) ou bien un binaire commun.

## La règle iOS / Android pour Google Sign-In

Subtilité contre-intuitive :

| | iOS | Android |
|---|---|---|
| Comment le SDK Google lit le server client id | **Info.plist** `GIDServerClientID` | Constructor arg en Dart |
| `GOOGLE_SERVER_CLIENT_ID` dart-define | Ignoré (Info.plist gagne) | **Obligatoire** sinon Google Sign-In silent fail |

Pour un IPA, oublier `GOOGLE_SERVER_CLIENT_ID` est sans conséquence. Pour un AAB, c'est un bug silencieux. D'où :

```ruby
# iOS lanes
require_mobile_env("REVENUECAT_IOS_KEY")

# Android lanes
require_mobile_env("REVENUECAT_ANDROID_KEY", "GOOGLE_SERVER_CLIENT_ID")
```

## Convention `.env` empty values

**Garder les lignes `KEY=` vides, ne pas les omettre.** C'est la convention dotenv officielle ([motdotla/dotenv#74](https://github.com/motdotla/dotenv/issues/74)) :

> "Empty value in .env = intentionally not set"

Une ligne `KEY=` documente "je sais que cette var existe, je n'en ai pas besoin localement". Si on l'omet, un nouveau dev en voyant `KEY=...` dans `.env.example` ne sait pas si c'est un oubli ou intentionnel.

## Le piège Bun auto-load `.env` dans les tests backend

Effets de bord d'avoir `KEY=` vide dans `apps/backend/.env` : Bun auto-charge ce fichier en `process.env` **avant** les preloads de test. Donc dans `test-setup.ts` :

```ts
// ❌ ??= ne déclenche pas sur empty string
process.env.REVENUECAT_WEBHOOK_SECRET ??= "test-default";  // reste ""

// ✅ ||= défaulte aussi sur empty
process.env.REVENUECAT_WEBHOOK_SECRET ||= "test-default";  // → "test-default"
```

**Utiliser `||=` (logical-or assign), pas `??=` (nullish-or assign)** dans le test setup quand le projet utilise dotenv. Cas réel : un `REVENUECAT_WEBHOOK_SECRET=` vide en dev `.env` faisait passer tous les tests RC en 401 dans le CI parce que les tests envoyaient un `Authorization:` vide.

## App version dans l'UI — jamais en dur

Pour afficher la version dans un écran "About" / Settings, **ne jamais
hardcoder** la string `1.0.0` dans le code Dart. À chaque bump tu vas
oublier de la mettre à jour, et un user va te ping avec une capture
"Settings dit 1.0.0 mais j'ai installé la 1.2.3".

Pattern correct : lire la version depuis le binaire au boot, l'injecter
via un provider Riverpod.

```dart
// apps/mobile/pubspec.yaml
dependencies:
  package_info_plus: ^9.0.1
```

```dart
// packages/features/profile/lib/src/providers/profile_providers.dart
class ProfileAppInfo {
  const ProfileAppInfo({required this.version, required this.buildNumber});
  final String version;
  final String buildNumber;
}

// Default placeholder — the mobile shell MUST override this. We pick
// "0.0.0" rather than throwing so widget tests don't need to
// override.
final profileAppInfoProvider = Provider<ProfileAppInfo>(
  (ref) => const ProfileAppInfo(version: '0.0.0', buildNumber: '0'),
);
```

```dart
// apps/mobile/lib/main.dart
import 'package:package_info_plus/package_info_plus.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final packageInfo = await PackageInfo.fromPlatform();

  final container = ProviderContainer(
    overrides: [
      // ...other overrides
      profileAppInfoProvider.overrideWithValue(
        ProfileAppInfo(
          version: packageInfo.version,        // pubspec `version: 1.0.0+1` → "1.0.0"
          buildNumber: packageInfo.buildNumber, // → "1"
        ),
      ),
    ],
  );
  runApp(UncontrolledProviderScope(container: container, child: ...));
}
```

```dart
// Settings screen
final appInfo = ref.watch(profileAppInfoProvider);
Text('${appInfo.version} (${appInfo.buildNumber})');
```

**D'où viennent les valeurs au runtime :**
- iOS : `CFBundleShortVersionString` (version) + `CFBundleVersion` (build) dans Info.plist, eux-mêmes générés depuis `$(FLUTTER_BUILD_NAME)` + `$(FLUTTER_BUILD_NUMBER)` qui sont remplis depuis `pubspec.yaml`
- Android : `versionName` + `versionCode` dans `android/app/build.gradle`, idem
- Le chemin entier : `pubspec.yaml: version: 1.0.0+1` → injecté dans Info.plist / build.gradle au build → `PackageInfo.fromPlatform()` les relit au runtime

Donc l'unique source de vérité est `pubspec.yaml`. Bump là, push tag, le binaire et l'UI s'auto-alignent.

**Piège à éviter** : si tu mets un fallback type `'0.0.0'` dans le default du provider et tu oublies de wirer le override dans `main.dart`, l'app affichera "0.0.0" silencieusement. Toujours grep `profileAppInfoProvider` après avoir ajouté un provider similaire pour vérifier qu'il a un override dans le shell.

## Symptômes à reconnaître

- **TestFlight build sans paywall** : `--dart-define` pas injecté dans le build Fastlane. Vérifier `flutter_build_ios` du Fastfile.
- **Google Sign-In marche en local, casse en prod Android** : `GOOGLE_SERVER_CLIENT_ID` manquant en dart-define. iOS bypass via Info.plist masque le bug.
- **Tests backend RC ou auth en 401 random** : `.env` dev a une var vide (`KEY=`), test-setup utilise `??=` → utiliser `||=`.
- **`flutter build` mange une dart-define avec espaces** : sed pattern fragile, utiliser le script `printf %q` au lieu d'un sed naïf.
- **Settings → About affiche "0.0.0"** : le provider d'app-version n'est pas overridé dans `main.dart`. Tu as oublié d'appeler `PackageInfo.fromPlatform()` au boot.

---

**Dernière vérif** : 2026-05-13 (Listable v1, audit pre-launch).
**Projets concernés** : `listable`.

Sources externes consultées :
- [Flutter — Continuous delivery](https://docs.flutter.dev/deployment/cd) (dart-define convention)
- [motdotla/dotenv #74](https://github.com/motdotla/dotenv/issues/74) (empty values convention)
- [Bun test runner](https://bun.com/docs/test) (preload pattern)
