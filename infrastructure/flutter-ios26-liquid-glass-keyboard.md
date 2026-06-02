# Flutter + iOS 26 Liquid Glass — coins blancs derrière le clavier

## Le symptôme

Sur iOS 26, quand le clavier s'ouvre au-dessus d'un écran avec un fond
NON-blanc (crème, teinté, sombre…), deux petits triangles **blancs**
apparaissent dans les coins arrondis en haut du clavier (juste au-dessus
de la barre de suggestions). Le reste de l'écran est correct ; seuls les
coins cutout du clavier montrent du blanc cru au lieu du contenu de la
page.

iOS 26 a rendu le clavier **translucide** (Liquid Glass) avec des coins
arrondis. Le verre composite les **pixels que Flutter peint** derrière
lui. Les coins cutout révèlent donc ce que Flutter rend à ces
coordonnées. Si Flutter n'y peint rien → on voit le blanc du framebuffer.

## La VRAIE cause (et les 6 fausses pistes)

La cause : avec `resizeToAvoidBottomInset: true` (le défaut), le Scaffold
**rétrécit son body au-dessus du clavier**. Dans le rect du clavier,
Flutter ne peint plus rien → blanc.

**Le piège mortel** : si l'écran est dans un `StatefulShellRoute` (tab
shell avec bottom nav), c'est le **Scaffold EXTERNE du shell** qui
resize. Mettre `resizeToAvoidBottomInset: false` sur le Scaffold de
l'écran interne ne sert **à rien** — la hauteur dispo est déjà coupée en
amont par le shell.

```
TabShell Scaffold (bottom nav)   ← resize:true par défaut = LE coupable
  └─ navigationShell
       └─ list_detail Scaffold   ← resize:false ici est INUTILE seul
```

Fausses pistes qu'on a toutes essayées (aucune ne marche) :

| Tentative | Pourquoi ça échoue |
|---|---|
| `UIWindow.backgroundColor = cream` (SceneDelegate) | Mauvaise window — le clavier vit dans une `UIRemoteKeyboardWindow` séparée. Et ça peint un bloc plat statique, pas le contenu vivant. |
| `Main.storyboard` / `LaunchScreen.storyboard` en crème | Le verre composite les pixels **Flutter**, pas la couleur native de la view. (Preuve : dian-dian a une storyboard BLANCHE et ses coins montrent quand même le rose du thème.) |
| `canvasColor: Colors.transparent` dans le ThemeData | Révèle le fond de la FlutterView, pas le contenu. Casse aussi Drawer/Dropdown. |
| `backgroundColor: Colors.transparent` sur le Scaffold de l'écran | Cause un **bleed** : pendant la transition de route go_router, le Scaffold transparent entrant révèle la page précédente. Et ne résout pas le blanc. |
| `UIDesignRequiresCompatibility = true` (Info.plist) | Marche mais c'est l'inverse de ce qu'on veut : ça **désactive** Liquid Glass → clavier opaque iOS-18. Solution de repli uniquement. |
| `resize:false` sur l'écran interne seul | Inutile si un Scaffold parent (tab shell) resize encore. |

## Le fix

`resizeToAvoidBottomInset: false` sur **TOUS** les Scaffolds dans la
chaîne, du shell jusqu'à l'écran — en gardant les Scaffolds **opaques**
(pas de `Colors.transparent` → pas de bleed de transition).

```dart
// 1. Le Scaffold du tab shell (StatefulShellRoute builder) — LE point clé
return Scaffold(
  resizeToAvoidBottomInset: false,
  body: navigationShell,
  bottomNavigationBar: NavigationBar(...),
);

// 2. Chaque écran qui ouvre un clavier sur un fond teinté
return Scaffold(
  resizeToAvoidBottomInset: false,
  appBar: AppBar(...),
  body: ...,  // le ListView/grid s'étend en pleine hauteur derrière le clavier
);
```

Résultat : le body s'étend en pleine hauteur, Flutter continue de peindre
le contenu (lignes de liste + fond crème entre elles) dans le rect du
clavier, et le verre Liquid Glass révèle ce contenu vivant dans ses coins
au lieu du blanc. **Du Dart pur → ça hot-reload**, pas de rebuild natif.

## Pourquoi opaque et pas transparent

Le réflexe (et tous les vieux SO sur "transparent corners") pousse vers
`backgroundColor: Colors.transparent` + un fond peint au root. Ça marche
chez dian-dian **parce qu'il n'utilise pas de routes** (écrans swappés via
`AnimatedSwitcher` sous un `CustomPaint` root persistant).

Avec **go_router**, chaque écran est une route. Un Scaffold transparent
entrant révèle la route sortante pendant la slide animation → bleed visible
("contenu de la page précédente visible 1 sec"). Inhérent, pas
contournable proprement avec des routes.

Un Scaffold **opaque** + `resize:false` peint exactement les mêmes pixels
dans le rect du clavier (fond du thème + contenu) que transparent +
root-paint, **sans le bleed**. Le verre du clavier montre le contenu
vivant pareil. Donc : opaque gagne avec go_router.

## Méthode de debug (l'ordre qui a fait gagner du temps à la fin)

Avant de bidouiller des couleurs, **éliminer les variables d'environnement** :

1. `flutter --version` sur les deux projets (le qui-marche et le
   qui-marche-pas). Même engine ? (Le support Liquid Glass dépend de
   l'engine.) → ici 3.44 des deux côtés, écarté.
2. `xcodebuild -version` + `-showsdks` : SDK iOS 26 présent ? → Xcode 26.2,
   écarté.
3. Diff des `Info.plist` (keys), deployment target, Podfile platform.
   → équivalents, écarté.
4. **Une fois l'environnement écarté, le delta est forcément dans l'arbre
   de widgets.** Comparer la structure Scaffold/route du projet qui marche.
   C'est là qu'on voit "transparent scaffolds + root paint, pas de routes"
   chez dian-dian → et qu'on remonte au Scaffold parent qui resize chez
   nous.

Le piège méta : on a passé des heures à recolorier des couches natives
(window, storyboard, canvas) en croyant à un problème de couleur. C'était
un problème de **layout** (qui resize quoi). Quand un fond "ne se peint
pas", chercher quel ancêtre coupe la hauteur, pas quelle couleur mettre.

## Repli si on doit shipper vite

Si pour une raison X le fix layout ne suffit pas (vieux Flutter sans le
support engine, etc.), opt-out de Liquid Glass :

```xml
<!-- ios/Runner/Info.plist -->
<key>UIDesignRequiresCompatibility</key>
<true/>
```

Clavier opaque iOS-18, zéro coin translucide donc zéro blanc. À retirer
dès que le fix layout marche, sinon on perd tout le look iOS 26.

## Symptômes à reconnaître

- **Coins blancs au clavier sur fond teinté, iOS 26** : un Scaffold
  parent resize. Mettre `resizeToAvoidBottomInset: false` sur TOUTE la
  chaîne (shell inclus).
- **"Page précédente visible 1 sec" à l'ouverture d'un écran** : un
  Scaffold est `backgroundColor: Colors.transparent` + go_router. Le
  remettre opaque.
- **Le fix marche pas alors que tu as mis resize:false** : tu l'as mis sur
  l'écran mais pas sur le `StatefulShellRoute`/tab shell parent.
- **Le champ focus passe sous le clavier** : trade-off de `resize:false` —
  le Scaffold ne scrolle plus auto le champ au-dessus du clavier. Si un
  TextField est en bas, le remonter manuellement (Scrollable.ensureVisible
  sur focus) ou accepter selon le layout.

---

**Dernière vérif** : 2026-06-02 (Listable v1, iOS 26.2, Flutter 3.44, Xcode 26.2).
**Projets concernés** : `listable`. Référence qui-marche : `dian-dian`.

Sources externes consultées :
- [flutter/flutter #179482](https://github.com/flutter/flutter/issues/179482) — Semi-transparent keyboard on iOS 26 reveals widgets that do not draw under it
- [flutter/flutter #170310](https://github.com/flutter/flutter/issues/170310) — body does not extend into keyboard rect
- [Scaffold.resizeToAvoidBottomInset](https://api.flutter.dev/flutter/material/Scaffold/resizeToAvoidBottomInset.html)
- [Apple — UIDesignRequiresCompatibility](https://developer.apple.com/documentation/BundleResources/Information-Property-List/UIDesignRequiresCompatibility)
