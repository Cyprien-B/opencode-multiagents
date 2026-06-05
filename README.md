# opencode-multiagents

Architecture multi-agents pour opencode, orchestrée par un agent
**Orchestrator** qui délègue toute tâche concrète à des sous-agents spécialisés.

## Workflow

> 📊
> [Voir le diagramme d'architecture](https://excalidraw.com/#json=fdpqOixa3J94liPN3j06d,XW-DtYDLACyoyRGYHlSLQg)

1. **Orchestrator** analyse la demande utilisateur et élabore un plan
2. Il délègue l'implémentation à `artisan` (pro) ou `artisan-lite` (flash)
3. Il peut lancer `explorer` pour cartographier le code ou `readerDoc` pour lire
   la doc
4. Après implémentation, `reviewer` et/ou `reviewerminimax` auditent le diff
5. En cas de désaccord bloquant, `reviewerArbiter` tranche
6. Une fois validé, `writerDoc` met à jour la documentation

## Agents principaux

| Agent            | Modèle              | Description                                                                                        |
| ---------------- | ------------------- | -------------------------------------------------------------------------------------------------- |
| **general**      | `kimi-k2.6`         | Agent primaire par défaut, utilisé directement par l'utilisateur.                                  |
| **Orchestrator** | `kimi-k2.6`         | Stratège pur — conçoit le plan, délègue toute exécution aux sous-agents, ne touche jamais au code. |
| **Cheapbuild**   | `deepseek-v4-flash` | Senior Dev à usage direct pour des tâches rapides sans passer par l'orchestrateur.                 |

## Sous-agents de l'Orchestrator

| Agent               | Modèle              | Description                                                                                              |
| ------------------- | ------------------- | -------------------------------------------------------------------------------------------------------- |
| **artisan**         | `deepseek-v4-pro`   | Implémenteur Pro — tâches complexes, multi-fichiers, architecturales ou sensibles.                       |
| **artisan-lite**    | `deepseek-v4-flash` | Implémenteur Flash — tâches étroites ou de complexité moyenne, moins coûteux que `artisan`.              |
| **explorer**        | `deepseek-v4-flash` | Navigateur lecture-seule — explore le codebase, trouve symboles et flux de données.                      |
| **readerDoc**       | `deepseek-v4-flash` | Analyste de documentation — extrait les exigences et conventions des fichiers de doc.                    |
| **reviewer**        | `qwen3.7-plus`      | Spécialiste QA (Qwen) — audite les diffs pour bugs, régressions et failles de sécurité.                  |
| **reviewerminimax** | `minimax-m3`        | Spécialiste QA (MiniMax) — seconde relecture en parallèle pour détecter ce qu'un seul reviewer raterait. |
| **reviewerArbiter** | `MiMo-V2.5-Pro`     | Arbitre de revue — départage les désaccords substantiels entre les deux reviewers.                       |
| **writerDoc**       | `deepseek-v4-flash` | Rédacteur technique — met à jour la documentation, le changelog et les docstrings après validation.      |

## Workflow auto de maintenance (cron quotidien)

Un cron `no_agent` tourne chaque jour à 12:00 et vérifie les limites
OpenCode Go sur `https://opencode.ai/docs/go/`. Si quelque chose change
dans la table des modèles, il applique automatiquement ce workflow:

| Cas détecté                              | Action auto                                                                                       |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Un modèle utilisé ici **disparaît**      | Remplacement par le modèle de même famille le plus récent encore listé dans l'abo OpenCode Go, puis commit + push sur `main` |
| Un nouveau modèle apparaît dans l'abo    | Recherche web exhaustive (4-6 sources) + rapport Telegram avec proposition d'intégration, **aucune modif de la config** tant que tu n'as pas validé |
| Le push GitHub échoue                    | Patch local conservé + alerte Telegram, retry au prochain cron (pas de rollback)                  |
| Modif cosmétique (juste URLs/endpoints)  | Notification neutre, pas d'action sur la config                                                  |

**Sync bidirectionnel:** au début de chaque run, le cron fait un
`git pull --ff-only` depuis `origin/main` (stash + pop automatique) pour
récupérer les éventuelles modifs push depuis une autre machine. Si le
pull échoue (conflit, divergence), le cron avorte avec alerte et ne fait
rien d'autre — pas de patch sur état stale.

**Doc auto-sync:** à chaque patch `opencode.json` (auto-fix), le cron
met aussi à jour la colonne `Modèle` du tableau dans `README.md`.
Seule la colonne modèle est touchée — les descriptions FR custom
que tu as écrites à la main sont préservées. Le commit est groupé
(`opencode.json` + `README.md` dans le même commit, message
`auto: replace <old> by <new> (N agents) + README sync`).

Détails, conventions, et module structure:
[`~/.hermes/skills/devops/provider-model-availability-monitor/SKILL.md`](https://github.com/Cyprien-B/opencode-multiagents)
(skill `provider-model-availability-monitor`, section "Reference
watchdog: opencode_go_check.py").
