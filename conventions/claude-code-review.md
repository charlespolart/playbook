# Code review auto par un subagent

**Convention transverse :** avant de conclure un tour où l'instance a édité du code non trivial, elle invoque un reviewer en subagent sur le `git diff` non commité. Le résultat : critiques bloquantes corrigées avant de rendre la main, warnings relayés au user en condensé.

## Le bon agent : `pr-review-toolkit:code-reviewer`

```
Agent(subagent_type: "pr-review-toolkit:code-reviewer", prompt: "<fichiers édités + résumé du changement>")
```

- Modèle **opus**, conçu pour relire proactivement le diff *avant* de déclarer une tâche finie (exactement notre besoin).
- Lit par défaut les changements non staged (`git diff`). On lui passe quand même la liste des fichiers + un résumé court du plan, sinon il devine.
- Vient du plugin `pr-review-toolkit@claude-plugins-official`. L'activer dans `~/.claude/settings.json` (`enabledPlugins`) ou via `/plugin`.

## Les pièges (vérifiés 2026-06-06)

1. **`superpowers:code-reviewer` n'existe pas.** Le plugin superpowers ne ship **aucun** dossier `agents/`. Une vieille consigne pointait dessus → `Agent type not found`. Ne pas l'utiliser.
2. **`/code-review` n'est pas un reviewer inline.** Ce plugin (`code-review@…`) est un **bot de review de PR GitHub** : il lit une pull request et poste un `gh pr comment`. Inutilisable sur un repo **master-only / sans PR**. Son mode local `ultra` (review de branche) est **déclenché par le user + facturé** → une instance ne peut pas le lancer elle-même.
3. **Un plugin fraîchement activé ne charge son agent qu'au prochain démarrage de session.** Si `pr-review-toolkit:code-reviewer` renvoie `not found`, c'est ça → **fallback sur `general-purpose`** avec les mêmes instructions de review, le temps que le user relance.

## Skills vs subagents — ce qui est auto, ce qui ne l'est pas

C'est *la* distinction à retenir (et la raison pour laquelle seul le reviewer a besoin d'une fiche) :

| Capacité | Type | Auto ? | À préciser par instance ? |
|---|---|---|---|
| **superpowers** (brainstorming, plans, TDD, debug) | skills + hook `SessionStart` | **Oui** — le hook injecte `using-superpowers` à chaque session et *force* l'invocation des skills pertinents | Non |
| **frontend-design** | skill | **Oui (quand pertinent)** — auto-surfacé, invoqué sur du front distinctif | Non |
| **code review** | **subagent** (tool `Agent`) | **Non** — aucun hook/skill n'invoque un subagent automatiquement | **Oui** — règle explicite dans CLAUDE.md + cette fiche |

Autrement dit : les *skills* et les *hooks* se déclenchent seuls ; les *subagents* doivent être appelés explicitement. Donc on documente le reviewer, pas superpowers ni design.

## Où le câbler dans un projet

Dans le `CLAUDE.md` du projet, une section « Reviewer auto » qui donne : le déclencheur (édition de code non trivial dans `apps/`/`packages/`, hors formatage/docs/générés), le `subagent_type` exact, et les exceptions (modifs < 5 lignes, renommages mécaniques, `*.md`/`assets/`, fichiers générés `*.g.dart`/`*.freezed.dart`).

## Doc officielle

- Subagents : https://docs.claude.com/en/docs/claude-code/sub-agents
- Plugins : https://docs.claude.com/en/docs/claude-code/plugins

---
*Dernière vérif : 2026-06-06 (Claude Code, plugins claude-plugins-official).*
