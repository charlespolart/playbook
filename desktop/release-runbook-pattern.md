# Script de release desktop — test par défaut, publication confirmée

Le problème à figer : une release d'app desktop, c'est **un seul script** qui
builde, signe, notarise, zippe, publie et pousse l'appcast — et une fois
publiée, **toute la base installée la reçoit**. Le script doit donc être
impossible à déclencher par réflexe, et sûr à rejouer quand une étape
tombe. Pattern vécu sur **Mumo** (`scripts/package-release.sh` + `Makefile`,
runbook `docs/RELEASING.md`).

## Deux modes, un seul script

| Commande | Fait | Qui reçoit |
|---|---|---|
| `make test-dmg` = `package-release.sh` | DMG signé + notarisé, **rien d'autre** | personne, rien en ligne |
| `make release` = `package-release.sh --publish` | idem **puis** zip signé + GitHub Release + appcast | **tous** les utilisateurs installés |

**Le défaut ne publie jamais.** Le DMG de test est identique à celui de la
vraie release (même signature, même notarisation) : on teste exactement ce que
les gens installeront.

## Trois confirmations tapées, pas cliquées

`make release` demande dans l'ordre : « as-tu incrémenté la version ? » →
taper `oui` ; « retape le numéro exact » → `0.3.0` ; puis le script lui-même
demande `publier`. La moindre saisie fausse annule **sans rien construire**.
Conséquence assumée : un agent sans clavier (Claude Code) ne peut pas publier
par accident ; il peut le faire sciemment avec
`printf 'oui\n0.3.0\npublier\n' | make release`, après accord explicite.

## Les garde-fous automatiques (dans `make check` ET dans le script)

Dans l'ordre, avant le long build pour ne pas découvrir le problème cinq
minutes plus tard :

1. `gh auth status` (droit d'écriture sur le dépôt des releases).
2. Certificat Developer ID présent (`security find-identity`).
3. Clé privée Sparkle au trousseau :
   `security find-generic-password -s 'https://sparkle-project.org' -a 'ed25519'`.
4. **Notes de version complètes** : `changelog.py --check <version>` refuse si
   une des langues livrées n'a pas au moins une puce.
5. **Anti-oubli-de-bump**, comparé à l'appcast **en ligne** (`curl` du feed) :
   refuse si `CFBundleShortVersionString` y figure déjà, ou si
   `CFBundleVersion` ≤ le plus grand `<sparkle:version>` publié. Appcast
   injoignable → laisse passer avec une notice (première release, réseau).

À ajouter au preflight après le vécu du 2026-11-04 : `xcrun metal --version`
(composant Metal manquant après une mise à jour d'Xcode) et un seuil d'espace
disque (cf. `macos-distribution-hors-store.md`, pièges 1 et 2).

## Idempotence — rejouable après une étape tombée

- Release GitHub déjà présente → `gh release upload --clobber`, pas de
  doublon.
- Version déjà listée dans l'appcast → aucun second item.
- La version publiée est lue dans l'**Info.plist embarqué** de l'app exportée,
  pas dans le fichier source : on publie ce qui est signé.
- Le script clone le dépôt des releases dans un dossier temporaire, insère
  l'item, copie `CHANGELOG.md`, committe `release: appcast <version>`, pousse.

## Source de vérité de la version

`Sources/<App>/Resources/Info.plist` et rien d'autre : `CFBundleShortVersionString`
(SemVer visible) + `CFBundleVersion` (entier, +1 à chaque release, sans
exception). L'écran « À propos » lit `Bundle.main.infoDictionary`, jamais une
chaîne en dur (même règle que `mobile/flutter-mobile-build.md`). Piège de
lecture : `defaults read <chemin>/Info` **sans** l'extension `.plist`.

Le `CHANGELOG.md` suit une grammaire stricte, `## <version> — <date>` /
`### <langue>` / `#### new|improved|fixed` / puces, parce que trois
consommateurs le lisent (fenêtre Sparkle, GitHub Release, fenêtre Nouveautés
de l'app). Une section `## Unreleased` accumule pendant le cycle et devient la
section de la version au moment du bump.

## Vérifier après coup (la sentinelle du desktop)

Comme la sentinelle du deploy serveur (`infrastructure/github-actions-deploy.md`
piège #9), une publication « verte » ne prouve rien tant qu'on n'a pas :
- `curl -s <feed>` → l'item de la version est là ;
- `curl -fsSI <enclosure url>` → 200/302 (les assets GitHub peuvent mettre
  quelques secondes) ;
- une app installée en version N−1 **propose et installe** N (menu
  « Rechercher les mises à jour… »).
Le premier point n'est pas encore automatisé dans le script de Mumo ; à faire.

## Symptômes à reconnaître

- `make release` s'arrête avant de builder → une saisie ne correspond pas,
  c'est voulu ; relire le message.
- « v0.3.0 est DÉJÀ publiée » alors qu'on vient de bumper → le bump n'a pas
  touché `CFBundleVersion`, ou pas été sauvegardé.
- Le script publie un numéro différent de celui attendu → Info.plist source
  et app exportée divergent (projet non régénéré) : relancer depuis
  `xcodegen generate`.
- `gh release create` en 403 → `gh` connecté sur un autre compte ou sans
  droit sur le dépôt des releases.

---

**Dernière vérif** : 2026-11-04 (première publication réelle de Mumo,
0.3.0 ; preflight Metal/disque identifié ce jour, pas encore scripté).

Sources externes :
- [Apple — Information Property List : CFBundleVersion](https://developer.apple.com/documentation/bundleresources/information-property-list/cfbundleversion)
- [GitHub CLI — gh release](https://cli.github.com/manual/gh_release)
- [Sparkle — Publishing an update](https://sparkle-project.org/documentation/publishing/)
