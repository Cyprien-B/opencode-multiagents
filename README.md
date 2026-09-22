# opencode-multiagents

Architecture multi-agents pour OpenCode, pilotée par un agent **Orchestrator** qui clarifie la demande, planifie le travail et coordonne des agents spécialisés dans des sessions réutilisables.

La configuration cible OpenCode **1.18.32** et conserve provisoirement les modèles et le provider `opencode-go`. La migration des sous-agents vers OpenRouter sera réalisée séparément.

## Workflow

> 📊
> [Voir le diagramme d'architecture](https://excalidraw.com/#json=fdpqOixa3J94liPN3j06d,XW-DtYDLACyoyRGYHlSLQg)

Deux points d'entrée sont disponibles :

- **Orchestrator** pour les demandes suffisamment définies et l'exécution complète du workflow ;
- **BigBrain** pour les projets encore flous qui nécessitent un entretien approfondi et une documentation de cadrage avant l'exécution.

Le workflow principal suit cet ordre :

1. **Orchestrator** consulte la demande, le projet et la documentation existante. Il pose uniquement les questions qui changent réellement l'implémentation, avec un maximum de **10 questions par demande** et une recommandation pour chacune.
2. Il crée un brief, découpe le travail en tâches, choisit `artisan` ou `artisan-lite`, attribue des périmètres d'écriture exclusifs et organise les vagues d'exécution.
3. L'artisan implémente la tâche et effectue ses vérifications locales. En cas de blocage technique persistant, il peut appeler **Helper**.
4. **reviewer** audite le diff réel après l'implémentation. Il couvre à la fois la correction, les régressions, la maintenabilité et la sécurité.
5. Lorsque la revue accepte la révision courante, **Orchestrator appelle lui-même `testeur`**. Le testeur exécute les contrôles indépendamment des conclusions de l'artisan et du reviewer.
6. Une correction revient au même artisan, puis au même reviewer, puis au même testeur. Toute modification invalide les validations de la révision précédente.
7. Après validation de toutes les tâches, **writerDoc** met à jour uniquement la documentation devenue inexacte ou manquante.

La boucle de validation est donc :

```text
artisan → reviewer → testeur
   ↑          │          │
   └──────────┴──────────┘ correction puis nouvelle révision
```

Le testeur n'est jamais appelé par un artisan, Helper, reviewer ou writerDoc. Cette séparation limite le risque qu'il adopte sans vérification les conclusions de l'agent qui a écrit ou relu le code.

## BigBrain

BigBrain adapte le fonctionnement de `grill-with-docs`, `grilling` et `domain-modeling` :

- entretien par séries de questions dépendantes avec réponses recommandées ;
- vérification dans le dépôt des faits qui peuvent être trouvés sans interroger l'utilisateur ;
- clarification du vocabulaire métier et mise à jour de `CONTEXT.md` lorsque des termes sont réellement stabilisés ;
- création parcimonieuse d'ADR pour les décisions difficiles à inverser, surprenantes sans contexte et issues d'un véritable compromis ;
- production d'un handoff dans `docs/workflow/<slug>/brief.md` avec périmètre, décisions, critères d'acceptation, contraintes, cas limites et questions restantes.

BigBrain ne lance jamais Orchestrator. Lorsque son handoff est prêt, l'utilisateur sélectionne Orchestrator et lui transmet le chemin du brief.

La version légère intégrée à Orchestrator reprend la recherche des faits, l'arbre de décisions et les recommandations, avec un plafond de 10 questions pour ne pas alourdir les tâches simples. S'il reste trop de décisions bloquantes après ce plafond, Orchestrator signale les zones d'ombre et propose de passer par BigBrain.

## Sessions réutilisables

Chaque workflow possède un état local dans :

```text
.opencode/workflow/<workflow_id>/
├── brief.md
└── ledger.json
```

`ledger.json` conserve notamment :

- les tâches, dépendances, critères d'acceptation et périmètres de lecture/écriture ;
- la révision courante de chaque tâche ;
- le compteur de questions ;
- les identifiants de sessions des artisans, reviewers, testeurs et rédacteurs ;
- les blocages, tentatives, validations et événements du workflow.

Lors du premier appel à un agent, Orchestrator omet `task_id`, puis enregistre l'identifiant retourné par OpenCode. Pour une correction ou une nouvelle vérification de la même tâche, il réutilise ce `task_id`. Les sessions restent disponibles lorsqu'un agent termine son tour : l'état de l'agent devient inactif, sans suppression de sa conversation.

Chaque rôle possède sa propre session. L'identifiant d'un artisan n'est jamais utilisé pour un reviewer ou un testeur. Un même agent ne reçoit pas deux appels simultanés dans la même session. Les sessions concernent un travail précis et ne sont pas réutilisées pour un projet sans rapport.

Si OpenCode ne retrouve plus une session, Orchestrator enregistre son remplacement et reconstitue explicitement le contexte à partir du brief, du registre, des rapports et des fichiers actuels. Les identifiants de sessions sont locaux à l'installation OpenCode : versionner le registre ne transporte pas la conversation vers une autre machine.

## Helper

Seuls `artisan` et `artisan-lite` peuvent invoquer Helper. La profondeur `subagent_depth: 2` permet la chaîne suivante :

```text
Orchestrator → artisan → Helper
```

Helper intervient :

- après trois approches distinctes sans progrès sur le même blocage ;
- immédiatement si l'artisan identifie une incapacité technique qui nécessite un modèle plus puissant.

L'artisan lui transmet la tâche bornée, les fichiers autorisés, les erreurs, les tentatives précédentes et les critères d'acceptation, puis suspend ses propres écritures sur ce périmètre. Helper peut modifier le code dans le périmètre transféré. Après son intervention, l'artisan relit et vérifie le résultat avant de le remettre à Orchestrator.

Une tâche dispose d'un seul épisode Helper sans nouvelle décision utilisateur, avec au maximum trois approches distinctes. Helper ne sert pas à contourner une permission refusée, un secret absent, un modèle indisponible ou une décision produit manquante. Son résultat passe toujours par `reviewer`, puis `testeur`.

## Agents principaux

| Agent | Modèle | Description |
| --- | --- | --- |
| **general** | `opencode-go/kimi-k2.7-code` | Agent primaire utilisé directement pour du développement manuel hors workflow. |
| **Cheapbuild** | `opencode-go/deepseek-v4-flash` | Senior Dev économique utilisé directement pour des tâches rapides hors workflow. |
| **BigBrain** | `opencode-go/kimi-k2.7-code` | Agent de cadrage approfondi ; interroge l'utilisateur et produit la documentation transmise ensuite à Orchestrator. |
| **Orchestrator** | `opencode-go/kimi-k2.7-code` | Coordinateur principal ; clarifie, planifie, délègue, conserve les sessions et applique les portes de validation. |

`general` et `Cheapbuild` ne participent pas au workflow orchestré et ne peuvent pas invoquer les autres agents.

## Agents du workflow

| Agent | Mode | Modèle | Description |
| --- | --- | --- | --- |
| **artisan** | `all` | `opencode-go/deepseek-v4-pro` | Implémentation complexe, architecturale, sensible ou multi-fichiers. Peut invoquer uniquement Helper. |
| **artisan-lite** | `all` | `opencode-go/deepseek-v4-flash` | Implémentation précise et limitée, normalement jusqu'à deux fichiers et environ 150 lignes nettes. Peut invoquer uniquement Helper. |
| **Helper** | `all` | `opencode-go/glm-5.3` | Reprise bornée d'un blocage technique ; ne délègue à aucun autre agent. |
| **reviewer** | `all` | `opencode-go/qwen3.7-plus` | Revue indépendante unique couvrant correction, régressions, maintenabilité et sécurité. |
| **testeur** | `all` | `opencode-go/deepseek-v4-flash` | Exécute les validations indépendantes après l'acceptation du reviewer ; lecture seule sur le code. |
| **writerDoc** | `all` | `opencode-go/deepseek-v4-flash` | Met à jour la documentation après validation sans modifier la logique source. |

Les agents `explorer`, `readerDoc`, `verifier`, `reviewerminimax` et `reviewerArbiter` ont été retirés du workflow. Les entrées encore nécessaires pour neutraliser d'anciens agents fusionnés sont déclarées avec `disable: true` dans `opencode.json` et ne constituent pas des rôles actifs.

Les agents du workflow utilisent le mode `all` afin de pouvoir être invoqués comme sous-agents par Orchestrator ou directement par la CLI avec `--agent`. Leurs permissions de délégation restent limitées par la configuration.

## Sélection artisan / artisan-lite

Orchestrator utilise `artisan-lite` par défaut lorsque la tâche est claire, limitée et sans risque particulier. Il sélectionne directement `artisan` lorsque l'une de ces conditions s'applique :

- au moins trois fichiers ou environ plus de 150 lignes nettes ;
- plusieurs préoccupations logiques ou synthèse entre plusieurs modules ;
- décision architecturale ou modification d'une API publique ;
- concurrence, sécurité, gestion d'erreur délicate, migration de données ou performance ;
- authentification, cryptographie, secrets ou entrées non fiables ;
- `artisan-lite` constate que le périmètre dépasse sa mission.

Chaque tâche mentionne le niveau choisi et sa justification. Les tâches indépendantes peuvent utiliser différents niveaux dans la même vague.

## Planification et parallélisme

Pour un travail non trivial, le plan indique l'objectif, l'approche, les hypothèses, les tâches, les agents choisis, les fichiers lus et écrits, les dépendances, les vagues, les tests, les risques, les éléments hors périmètre et la répartition des modèles.

Une vague contient au maximum quatre implémenteurs. Deux tâches ne sont parallélisées que si leurs écritures sont disjointes, qu'elles n'ont aucune dépendance lecture-après-écriture, aucun état mutable partagé, aucun ordre sémantique et aucune collision de tests.

Les installations de dépendances, lockfiles, migrations, générations de code, formatages massifs, bases partagées et runners utilisant le même port sont exécutés séquentiellement. Si un agent touche un fichier hors de son périmètre ou si deux agents modifient le même fichier, la vague suivante est suspendue et le plan est corrigé.

Une tâche simple et complètement définie peut être exécutée avec un plan court. Les tâches complexes conservent un plan détaillé. Une demande de planification seule s'arrête avant l'implémentation.

## Bugs et régressions

Lorsqu'un problème concerne une régression ou un comportement inattendu, Orchestrator clarifie d'abord la reproduction. Si la cause n'est pas déjà établie, il utilise `reviewer` en mode diagnostic pour retrouver la cause dans le code et l'historique.

Le diagnostic ne vaut pas validation du correctif. La correction est confiée à un artisan, puis suit la boucle habituelle :

```text
diagnostic → artisan → reviewer → testeur
```

## Utilisation depuis Claude Code ou Codex

L'objectif futur est de permettre à Claude Code ou Codex de remplacer Orchestrator tout en conservant les travailleurs OpenCode facturés par leur provider configuré.

- Un utilisateur Claude conserve son abonnement dans le binaire officiel Claude Code. Claude Code appelle OpenCode en ligne de commande et n'envoie aucun jeton d'abonnement Claude à OpenCode.
- Un utilisateur OpenAI peut utiliser directement Orchestrator dans OpenCode avec son modèle principal préféré lorsque la connexion le permet.
- Les sous-agents pourront être configurés sur OpenRouter dans une étape séparée afin d'utiliser les crédits OpenRouter de chaque membre de l'équipe.

Exemple de premier appel direct d'un travailleur :

```bash
opencode run --agent artisan --format json --file mission.md "Exécutez la mission jointe en mode headless et retournez le rapport structuré."
```

Le coordinateur externe extrait l'identifiant de session du flux JSON, puis le réutilise :

```bash
opencode run --agent artisan --session SESSION_ID_REEL --format json --file correction.md "Reprenez la même tâche avec cette correction."
```

Le coordinateur externe doit appliquer le même contrat : une seule instance coordonne le workflow et écrit le registre, les missions contiennent le contexte complet, les appels sont sérialisés par session, les corrections réutilisent les identifiants, et chaque nouvelle révision repasse par reviewer puis testeur.

Le skill Claude Code/Codex qui automatisera cette intégration n'est pas encore fourni dans ce dépôt.

## Installation et vérification

Placer `opencode.json` à la racine du projet, puis vérifier la configuration :

```bash
opencode --version
opencode agent list
opencode debug config
```

OpenCode fusionne les configurations du projet et de l'utilisateur. Les anciennes définitions dans `.opencode/agents/` ou dans la configuration personnelle peuvent donc influencer le résultat. Les rôles supprimés sont désactivés explicitement dans le fichier livré.

Le dossier `.opencode/workflow/` contient un état opérationnel local. Il devrait généralement être exclu de Git. Les briefs BigBrain placés dans `docs/workflow/` sont des documents projet et peuvent être versionnés.

## Limites actuelles

- Les modèles utilisent encore `opencode-go/...` ; les crédits OpenRouter ne sont pas encore consommés par cette configuration.
- `opencode-go/glm-5.3` est l'identifiant demandé pour Helper ; sa disponibilité dépend du catalogue du provider.
- La réutilisation des sessions est fournie par `task_id`, mais les compteurs, le registre et l'ordre des phases sont appliqués par les prompts.
- Le mode CLI direct permet à un humain de sélectionner un agent `all` ; `opencode.json` ne constitue pas un système d'authentification de l'appelant.
- Les accès shell des agents capables d'exécuter des tests peuvent produire des caches et artefacts de build.

## Workflow auto de maintenance (cron quotidien)

Un cron `no_agent` tourne chaque jour à 12:00 et vérifie les limites OpenCode Go sur `https://opencode.ai/docs/go/`. Si quelque chose change dans la table des modèles, il applique automatiquement ce workflow :

| Cas détecté | Action automatique |
| --- | --- |
| Un modèle utilisé ici **disparaît** | Remplacement par le modèle de même famille le plus récent encore listé dans l'abonnement OpenCode Go, puis commit et push sur `main`. |
| Un nouveau modèle apparaît dans l'abonnement | Recherche web approfondie et rapport Telegram avec proposition d'intégration ; aucune modification de la configuration avant validation. |
| Le push GitHub échoue | Patch local conservé, alerte Telegram et nouvel essai au prochain cron, sans rollback. |
| Modification cosmétique des URL ou endpoints | Notification neutre, sans changement de la configuration. |

Au début de chaque exécution, le cron lance `git pull --ff-only` depuis `origin/main` avec stash/pop automatique afin de récupérer les modifications envoyées depuis une autre machine. En cas de conflit ou de divergence, il s'arrête avec une alerte afin de ne pas appliquer de patch sur un état obsolète.

Lorsqu'il modifie `opencode.json`, le cron met également à jour la colonne `Modèle` des tableaux de ce README. Les descriptions personnalisées restent intactes. Le commit regroupe `opencode.json` et `README.md` avec un message de la forme :

```text
auto: replace <old> by <new> (N agents) + README sync
```

Les détails du watchdog se trouvent dans le skill `provider-model-availability-monitor` du dépôt.
