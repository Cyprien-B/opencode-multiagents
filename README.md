# opencode-multiagents

Architecture multi-agents pour opencode, orchestrée par un agent **Orchestrator** qui délègue toute tâche concrète à des sous-agents spécialisés.

## Workflow

1. **Orchestrator** analyse la demande utilisateur et élabore un plan
2. Il délègue l'implémentation à `artisan` (pro) ou `artisan-lite` (flash)
3. Il peut lancer `explorer` pour cartographier le code ou `readerDoc` pour lire la doc
4. Après implémentation, `reviewer` et/ou `reviewerminimax` auditent le diff
5. En cas de désaccord bloquant, `reviewerArbiter` tranche
6. Une fois validé, `writerDoc` met à jour la documentation

## Agents principaux

| Agent | Modèle | Description |
|-------|--------|-------------|
| **general** | `kimi-k2.6` | Agent primaire par défaut, utilisé directement par l'utilisateur. |
| **Orchestrator** | `kimi-k2.6` | Stratège pur — conçoit le plan, délègue toute exécution aux sous-agents, ne touche jamais au code. |
| **Cheapbuild** | `deepseek-v4-flash` | Senior Dev à usage direct pour des tâches rapides sans passer par l'orchestrateur. |

## Sous-agents de l'Orchestrator

| Agent | Modèle | Description |
|-------|--------|-------------|
| **artisan** | `deepseek-v4-pro` | Implémenteur Pro — tâches complexes, multi-fichiers, architecturales ou sensibles. |
| **artisan-lite** | `deepseek-v4-flash` | Implémenteur Flash — tâches étroites ou de complexité moyenne, moins coûteux que `artisan`. |
| **explorer** | `deepseek-v4-flash` | Navigateur lecture-seule — explore le codebase, trouve symboles et flux de données. |
| **readerDoc** | `deepseek-v4-flash` | Analyste de documentation — extrait les exigences et conventions des fichiers de doc. |
| **reviewer** | `qwen3.6-plus` | Spécialiste QA (Qwen) — audite les diffs pour bugs, régressions et failles de sécurité. |
| **reviewerminimax** | `minimax-m2.7` | Spécialiste QA (MiniMax) — seconde relecture en parallèle pour détecter ce qu'un seul reviewer raterait. |
| **reviewerArbiter** | `MiMo-V2.5-Pro` | Arbitre de revue — départage les désaccords substantiels entre les deux reviewers. |
| **writerDoc** | `deepseek-v4-flash` | Rédacteur technique — met à jour la documentation, le changelog et les docstrings après validation. |