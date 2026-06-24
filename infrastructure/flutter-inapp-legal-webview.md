# Flutter — liens légaux en in-app WebView (pas Safari)

## Le piège

Sur mobile, tap sur "Privacy" / "Terms" / "Contact" via `launchUrl(uri, mode: LaunchMode.externalApplication)` éjecte l'utilisateur dans Safari/Chrome. Trois conséquences :

1. **Discontinuité visuelle.** Animation système, barre d'URL, partage, onglets — l'utilisateur a l'impression d'avoir quitté l'app. Pour les pages qu'on veut juste qu'il survole (CGU, mentions légales), c'est un over-kill qui casse l'illusion d'une app cohérente.
2. **Retour en arrière non garanti.** Sur iOS, il faut un back-tap sur la status bar (qui suppose que la "back-to-app" pill est affichée). Sur Android, c'est le back-system qui dépend du launcher. Une partie des users ne revient pas.
3. **Risque review Apple sur le paywall.** §3.1.2(a) exige Terms + Privacy sur l'écran d'abonnement. Apple n'interdit pas le hand-off Safari, mais une WebView in-app passe sans friction et limite les rejets "user can't read your terms before subscribing".

Un projet shippait `LaunchMode.externalApplication` partout (Settings + paywall) jusqu'au commit `0853616`.

## Le bon pattern

`webview_flutter` (officiel Flutter, maintenu par Google) + un helper partagé qui push une route fullscreen avec l'AppBar de l'app.

```dart
// packages/ui_kit/lib/src/components/in_app_web_page.dart
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

Future<void> showInAppWebPage(
  BuildContext context, {
  required Uri uri,
  String? title,
}) {
  return Navigator.of(context, rootNavigator: true).push(
    MaterialPageRoute<void>(
      fullscreenDialog: true,                       // slide-up sur iOS
      builder: (_) => _InAppWebPage(uri: uri, title: title ?? uri.host),
    ),
  );
}
```

Le `_InAppWebPage` State expose `WebViewController` avec `setJavaScriptMode(JavaScriptMode.unrestricted)`, `setBackgroundColor(AppColors.bg)` (évite le flash blanc avant le premier paint), et un `LinearProgressIndicator` en haut tant que `onPageFinished` n'a pas fired.

Une AppBar / TopBar app-branded (pas la `AppBar` Material par défaut) renforce le côté "tu n'es pas sorti de l'app".

## La règle de décision

| Type de lien | Comportement |
|---|---|
| Page légale self-hostée (Privacy, Terms, Legal, About, Contact) | `showInAppWebPage` |
| Mailto: (`mailto:support@…`) | `launchUrl(uri, mode: externalApplication)` |
| Tel: (`tel:+33…`) | `launchUrl(uri, mode: externalApplication)` |
| App Store / Play Store deep link (`apps.apple.com/account/subscriptions`) | `launchUrl(uri, mode: externalApplication)` |
| Lien externe vers un site tiers (blog, docs externes) | `launchUrl(uri, mode: externalApplication)` |

**Critère** : "ce lien est-il un document de mon écosystème que l'utilisateur doit juste lire et dismisser ?" → in-app. Sinon → handoff.

## Localisation des pages

Les apps servent leurs pages légales en plusieurs langues. Pattern partagé : un helper `localizedXxxUri(base, langCode)` qui transforme l'URL juste avant de la passer au WebView.

Une app utilise un préfixe path : `/legal/privacy` (fr) → `/en/legal/privacy` ; une autre un query param : `?lang=zh-CN`. Le helper se charge de l'aiguillage selon la convention backend de chaque app.

L'aiguillage doit rester dans le helper UI, pas dans le caller. Sinon chaque appelant duplique la map `Language → code`.

## Quand garder le hand-off externe (App Store sub management)

Pour la gestion d'abonnement, RC documente que la "manage subscription" page doit ouvrir l'app App Store via `https://apps.apple.com/account/subscriptions`. Si on l'ouvre en WebView, Apple intercepte parfois et redirige vers l'app, parfois pas (selon l'OS / la version). Garder `launchUrl(externalApplication)` ici — c'est explicitement un cas où le système doit gérer.

## Setup pubspec

```yaml
dependencies:
  webview_flutter: ^4.13.1
```

Si `webview_flutter` arrive déjà comme transitive (via `google_mobile_ads` p. ex.), **le promote en direct quand même** pour pinner la version et la documenter explicitement. Une transitive peut sauter de version majeure à n'importe quel `pub upgrade` sans qu'on le voie.

## Le piège SPM (iOS, début 2026)

`webview_flutter_wkwebview` 3.x est passé à Swift Package Manager. Si tu utilises aussi `google_mobile_ads` (encore CocoaPods uniquement à ce jour), pod install plante :

```
Unable to find a specification for `webview_flutter_wkwebview` depended upon by `google_mobile_ads`
A dependency conflict has occurred because google_mobile_ads uses CocoaPods
while webview_flutter_wkwebview uses Swift Package Manager.
```

Fix temporaire (recommandé par Flutter dans le message d'erreur) :

```bash
flutter config --no-enable-swift-package-manager
rm -rf ios/Runner.xcodeproj/project.xcworkspace/xcshareddata/swiftpm
flutter clean && flutter pub get
cd ios && rm -rf Pods Podfile.lock && pod install
```

À revoir quand Google Ads migre à SPM (issue tracker : <https://github.com/googleads/googleads-mobile-flutter/issues/725>).

## Symptômes à reconnaître

- **User tap "Terms" → ouvre Safari** : `launchUrl` au lieu de `showInAppWebPage`. Grep `LaunchMode.externalApplication`.
- **WebView fond blanc avant load** : pas de `setBackgroundColor` sur le controller. Le contenu pop visuellement quand le HTML arrive.
- **Pas de bouton close visible** : `Navigator.push` sans `fullscreenDialog: true` → l'écran arrive par la droite (push iOS standard) au lieu de slide-up, et iOS n'ajoute pas de croix automatique. Avec `fullscreenDialog`, il faut quand même mettre un `IconButton(Icons.close)` dans l'AppBar — pas auto.
- **`pod install` 401 sur webview_flutter_wkwebview** : conflit CocoaPods/SPM avec `google_mobile_ads`. Voir section dédiée ci-dessus.

## Références

- [`webview_flutter` 4.13.1 README](https://pub.dev/packages/webview_flutter) — configuration WebViewController
- [Apple Human Interface Guidelines — Modality](https://developer.apple.com/design/human-interface-guidelines/modality) — quand utiliser fullscreen dialog
- [Apple §3.1.2(a) subscriptions](https://developer.apple.com/app-store/review/guidelines/#3.1.2) — exigences Terms + Privacy sur paywall

---

**Dernière vérif** : 2026-06-02.
