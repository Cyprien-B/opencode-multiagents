# Workflow OpenCode — équipe à sessions réutilisables

Configuration préparée le 22 septembre 2026 pour **OpenCode 1.18.32**, dernière
version stable publiée lors de la vérification. Le fichier utilise le format V1
: `agent`, `prompt`, `permission`, `task` et `task_id`. Il ne mélange pas les
champs de la documentation V2.

## Installation

Placer `opencode.json` à la racine du projet, en remplacement de l'ancienne
configuration après sauvegarde. Aucun fichier de prompt, plugin, serveur MCP ou
skill externe n'est nécessaire : les instructions sont intégrées au JSON. Les
fichiers de contexte et de suivi sont créés par les agents au moment où ils
deviennent utiles.

Vérifier la version et les agents chargés :

```bash
opencode --version
opencode agent list
opencode debug config
```

OpenCode fusionne les configurations. D'anciens fichiers d'agents dans
`.opencode/agents/` ou dans la configuration personnelle peuvent donc
interférer. Les entrées `disable: true` neutralisent les anciens rôles et les
agents intégrés `build`, `plan` et `explore` ; elles ne constituent pas des
agents actifs. Les agents techniques internes de résumé/compaction restent gérés
par OpenCode.

## Rôles conservés

| Agent          | Mode    | Modèle de cette livraison       | Fonction                                                  |
| -------------- | ------- | ------------------------------- | --------------------------------------------------------- |
| `general`      | primary | `opencode-go/kimi-k2.7-code`    | Développement manuel hors workflow                        |
| `Cheapbuild`   | primary | `opencode-go/deepseek-v4-flash` | Développement manuel économique                           |
| `BigBrain`     | primary | `opencode-go/kimi-k2.7-code`    | Entretien approfondi et documentation de transmission     |
| `Orchestrator` | primary | `opencode-go/kimi-k2.7-code`    | Clarification légère, coordination, registre des sessions |
| `artisan`      | all     | `opencode-go/deepseek-v4-pro`   | Réalisation complexe ou sensible                          |
| `artisan-lite` | all     | `opencode-go/deepseek-v4-flash` | Réalisation précise et limitée                            |
| `Helper`       | all     | `opencode-go/glm-5.3`           | Reprise d'un blocage technique                            |
| `reviewer`     | all     | `opencode-go/qwen3.7-plus`      | Revue unique de correction et sécurité                    |
| `testeur`      | all     | `opencode-go/deepseek-v4-flash` | Vérification indépendante après revue                     |
| `writerDoc`    | all     | `opencode-go/deepseek-v4-flash` | Documentation après validation                            |

`explorer`, `readerDoc`, `verifier` et `reviewerminimax` sont remplacés et
désactivés. Le reviewer unique conserve le modèle du rôle `reviewer` précédent ;
son prompt couvre également la correction fonctionnelle. Le testeur reprend le
modèle du précédent `verifier`. BigBrain reprend provisoirement celui de
l'orchestrateur. L'identifiant Helper est écrit comme demandé ; sa disponibilité
chez le fournisseur n'a pas été testée par un appel facturé.

**Les fournisseurs ne sont pas migrés dans cette livraison.** Tous les
identifiants restent `opencode-go/...`. Le futur passage des travailleurs à
OpenRouter devra changer leurs champs `model` et connecter OpenRouter ; les
crédits OpenRouter ne sont donc pas encore consommés par ce fichier. Aucun
fallback silencieux vers le modèle personnel du coordinateur n'est prévu dans
les prompts.

## Déroulement

Entrée directe : sélectionner `Orchestrator` (agent par défaut) et décrire le
travail. Il consulte le projet, pose de zéro à dix questions au total par
demande, puis délègue. Chaque question comporte une recommandation ; les
réponses déjà connues ne sont pas redemandées. Une tâche simple suffisamment
claire ne requiert pas d'approbation rituelle supplémentaire.

Entrée approfondie : sélectionner `BigBrain`, préciser le projet ou la
fonctionnalité, répondre à l'entretien, puis confirmer la compréhension commune.
Il produit notamment `docs/workflow/<slug>/brief.md`. Sélectionner ensuite
`Orchestrator` et lui donner ce chemin. **BigBrain ne lance jamais
l'orchestrateur.**

Le comportement BigBrain est une adaptation intégrée de `grill-with-docs`,
`grilling` et `domain-modeling` : questions dépendantes, recommandations,
vérification des faits, glossaire et décisions d'architecture. Le skill amont
n'est pas installé ni prétendument chargé. L'adaptation remplace la découverte
par sous-agent par une lecture directe, puisque les agents de découverte ont été
supprimés. Elle ajoute un document de transmission complet, que le skill amont
ne garantit pas à lui seul.

La boucle de réalisation est :

1. Établissement de l'état de départ, puis attribution d'un périmètre d'écriture
   exclusif.
2. Travail de l'artisan, avec ses vérifications locales.
3. Revue indépendante par `reviewer`.
4. Si la revue accepte la révision courante, **l'orchestrateur seul** appelle
   `testeur`.
5. Toute correction retourne au même artisan, puis au même reviewer, puis au
   même testeur.
6. Après validation de l'ensemble, mise à jour utile de la documentation.

Un échec bloquant ou l'absence de test exécutable n'est jamais converti en
succès. Si la documentation modifie des commentaires source, des exemples testés
ou des entrées d'outillage, les validations concernées sont rouvertes. Une
modification de prose seule nécessite une vérification de cohérence.

Le testeur reçoit les exigences, le périmètre, l'état de départ et les commandes
factuelles. Il ne reçoit pas les conclusions rassurantes de l'artisan ou du
reviewer comme preuves. Cette séparation réduit le biais ; elle ne garantit pas
l'absence absolue de biais d'un modèle.

## Sessions, registre et échanges

L'orchestrateur crée `.opencode/workflow/<workflow_id>/brief.md` et
`ledger.json`. Il en est l'unique rédacteur. Deux coordinateurs ne doivent pas
gérer simultanément le même workflow.

Le registre conserve les tâches, révisions, exigences, périmètres d'écriture,
décisions, compteur de questions, identifiants des sessions, blocages et
validations. Les travailleurs communiquent par leurs résultats structurés ;
l'orchestrateur transmet les éléments utiles dans la session appropriée. Il n'y
a ni messagerie artisan/reviewer, ni serveur de boîtes aux lettres, ni boucle
d'attente.

À la première délégation, le coordinateur omet `task_id` et conserve
l'identifiant retourné par OpenCode. Lors d'une correction, il transmet ce même
identifiant. Il maintient des sessions distinctes pour chaque rôle et chaque
travail indépendant. Deux appels simultanés vers la même session sont interdits
par le protocole.

| Événement                                  | Comportement                                                                                          |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| L'artisan termine                          | Rapport `READY_FOR_REVIEW`, puis état inactif ; session conservée                                     |
| Le reviewer demande une correction         | Reprise du même artisan avec les constats utiles                                                      |
| Le code est corrigé                        | Nouvelle révision ; anciennes validations invalidées                                                  |
| La revue passe                             | Appel du testeur par le coordinateur                                                                  |
| Les tests échouent                         | Même artisan → même reviewer → même testeur                                                           |
| Le coordinateur reprend après interruption | Lecture du registre et rapprochement avec les fichiers actuels                                        |
| Un identifiant ne peut plus être chargé    | Détection d'un éventuel nouvel ID, événement `SESSION_REPLACED`, reconstitution explicite du contexte |
| Le travail est accepté                     | Tâche terminée, sessions conservées pour un suivi éventuel                                            |

Les sessions sont persistées dans les données locales d'OpenCode. Un registre
partagé dans Git ne transporte pas ces sessions vers la machine d'un collègue.
Les documents de transmission, eux, peuvent être versionnés. Le dossier
opérationnel `.opencode/workflow/` devrait généralement être exclu de Git selon
les conventions du projet ; la configuration ne modifie pas automatiquement
`.gitignore`.

L'exécution en arrière-plan n'est pas nécessaire. Les appels standards suffisent
; plusieurs tâches indépendantes peuvent être déléguées lorsque le client sait
exécuter les appels en parallèle. L'option expérimentale d'arrière-plan n'est
pas activée par ce fichier.

## Helper

Seuls `artisan` et `artisan-lite` peuvent appeler Helper via `task`.
`subagent_depth: 2` autorise la chaîne Orchestrator → artisan → Helper. Helper
ne peut déléguer à personne.

Après trois tentatives distinctes sans progrès sur un même blocage, ou
immédiatement si une incapacité technique est identifiée, l'artisan lui
transfère le périmètre concerné et suspend ses propres écritures. Helper reçoit
les erreurs, reproductions, tentatives précédentes et critères d'acceptation ;
il doit réaliser la correction quand c'est possible.

Une seule intervention de secours est permise par tâche sans nouvelle décision
utilisateur, avec au maximum trois approches distinctes dans cette intervention.
Les refus de permission, décisions produit manquantes, modèles indisponibles ou
identifiants absents ne doivent pas être contournés par Helper. Si le secours
échoue, le coordinateur expose le blocage. Si le secours réussit, revue et tests
restent obligatoires.

## Utilisation future depuis Claude Code ou Codex

L'architecture proposée est réalisable en séparant les facturations :

| Usage futur                                | Coordinateur                                                      | Travailleurs                                   |
| ------------------------------------------ | ----------------------------------------------------------------- | ---------------------------------------------- |
| Abonnement Claude personnel                | Claude Code officiel, connecté personnellement                    | OpenCode CLI connecté à OpenRouter             |
| Abonnement ChatGPT personnel dans OpenCode | Orchestrator OpenCode avec modèle OpenAI choisi par l'utilisateur | Agents configurés explicitement sur OpenRouter |
| Codex comme interface principale           | Codex, avec une intégration de délégation à préparer              | OpenCode CLI connecté à OpenRouter             |

Anthropic réserve l'authentification par abonnement à l'usage prévu de ses
applications natives ; la documentation OpenCode indique explicitement que les
plugins utilisant Claude Pro/Max dans OpenCode sont interdits par Anthropic. La
solution prévue garde donc Claude dans le binaire officiel et ne transmet aucun
jeton d'abonnement à OpenCode. Le fait que Claude Code appelle un outil externe
ne convertit pas son abonnement en crédits API : les travailleurs restent
facturés à leur fournisseur configuré.

OpenCode documente `/connect` → OpenAI → ChatGPT Plus/Pro. La liste effective
des modèles dépend du compte connecté. L'architecture est documentée ; les
authentifications personnelles et appels facturés n'ont pas été exercés ici.

Les travailleurs utilisent `mode: "all"` parce que la CLI 1.18.32 refuse de
sélectionner directement un profil strictement `subagent` avec `--agent`. Leurs
permissions `task` restent limitées. Ils peuvent donc être appelés directement
sans créer un nouvel Orchestrator à l'intérieur du workflow externe.

Exemples de commandes à exécuter depuis la racine du projet, avec un fichier de
mission réel préparé par le coordinateur :

```bash
opencode run --agent artisan --format json --file mission.md "Exécutez la mission jointe en mode headless et retournez le rapport du contrat."
```

Cette commande démarre la session artisan. Le coordinateur extrait l'identifiant
réellement renvoyé dans les événements JSON et le conserve avec le rapport. Le
fichier de mission doit préciser objectif, critères d'acceptation, fichiers
autorisés, état de départ et commandes de validation.

```bash
opencode run --agent artisan --session SESSION_ID_REEL --format json --file correction.md "Reprenez la même tâche à partir de la correction jointe en mode headless."
```

`SESSION_ID_REEL` est à remplacer par l'identifiant reçu, sans utiliser
`--continue` qui pourrait reprendre une autre session, ni `--fork` qui en
créerait une nouvelle. La sortie `--format json` est un flux d'événements ; le
rapport final de l'agent est du texte inclus dans ce flux. Une future
intégration doit analyser les événements, détecter erreurs et permissions
refusées, vérifier le changement éventuel d'ID et sérialiser les appels par
session.

Le futur skill Claude Code/Codex devra reprendre les règles de l'orchestrateur,
gérer le registre, fournir des missions complètes et appeler directement
artisan, reviewer, testeur et writerDoc. **Ce skill externe n'est pas livré ni
installé dans cette phase.** En mode non interactif, une information matérielle
manquante revient au coordinateur sous forme de `BLOCKED_INPUT`, au lieu
d'attendre un dialogue impossible.

## Portée des garanties

Les permissions de délégation sont configurées dans OpenCode : seul Orchestrator
autorise `task` vers testeur ; les artisans autorisent seulement Helper. Les
autres travailleurs interdisent toute délégation. Un utilisateur humain peut
toujours sélectionner manuellement un agent en mode `all` ; cette configuration
n'authentifie pas un appelant CLI.

Le registre, les plafonds de questions/tentatives, l'ordre des phases et les
périmètres d'écriture sont des instructions aux modèles, pas un moteur de
workflow déterministe. Les permissions d'édition et de délégation sont
appliquées par OpenCode ; un accès shell permettant des tests peut également
produire des fichiers. Pour une séparation hostile ou une orchestration
strictement garantie, il faudrait une couche d'exécution supplémentaire et des
environnements isolés. Aucune garantie de cette nature n'est prétendue ici.

La validation effectuée porte sur la lecture du JSON par le binaire officiel
1.18.32, la résolution des agents, leurs modes et permissions, et les invariants
de routage du fichier. Aucun développement complet ni appel de modèle payant n'a
été exécuté. Une installation avec des plugins ou configurations supplémentaires
peut modifier le résultat effectif.
