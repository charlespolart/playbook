# Flutter : `fontFamily: 'monospace'` ne résout pas sur iOS

## Symptômes à reconnaître

- Un code / compteur / prix stylé `fontFamily: 'monospace'` s'affiche bien sur
  Android mais retombe sur la police par défaut sur iOS — l'alignement en
  colonnes que le mono devait garantir disparaît.
- Des chiffres qui « dansent » : un compteur `3/12 → 4/12`, un chrono `mm:ss`
  ou une colonne de prix dont la largeur bouge à chaque tick/frappe.

## Le piège

`'monospace'` est un **alias Android** ; iOS n'a pas de famille de ce nom et
Flutter retombe silencieusement sur la police du thème (pas d'erreur, pas de
warning). Tout code qui compte sur ce nom pour aligner des glyphes est cassé
sur la moitié du parc.

## La bonne façon

Dans 95 % des cas, on ne voulait pas du *mono* — on voulait des **chiffres à
chasse fixe**. La police du design system (Inter ici) les fournit via une
feature OpenType, sans changer de fonte :

```dart
style: theme.textTheme.bodyMedium?.copyWith(
  fontFeatures: const [FontFeature.tabularFigures()],
),
```

→ tous les chiffres font la même largeur (compteurs, prix, codes numériques,
chronos stables), la typo reste celle du design system, et ça marche partout.

Si on veut *vraiment* du monospace (bloc de code) : embarquer une vraie fonte
mono en asset (`JetBrainsMono`…), jamais l'alias `'monospace'`.

Règle de design associée (validée sur Listable) : **tout nombre qui change ou
s'aligne en colonne = tabular figures** — compteurs de listes, totaux €,
countdown OTP, positions d'ordre de rayon.

Doc : <https://api.flutter.dev/flutter/dart-ui/FontFeature/FontFeature.tabularFigures.html>

---

**Dernière vérif** : 2026-07-12 (7 occurrences corrigées sur Listable — codes
d'invitation, compteurs, prix).
