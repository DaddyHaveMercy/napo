# AiDPGen — Documentation d'architecture

## Vue d'ensemble

AiDPGen (Agentic Data Pipeline Generator) est un agent qui automatise le traitement de tickets JIRA de création de pipelines de données. Il enveloppe le générateur CTDF v2 existant dans une couche d'orchestration agentique : à partir d'un ticket, il récupère la configuration depuis Confluence, génère le SQL, l'exécute sur Starburst, et valide les résultats — avec des points de contrôle humains (Gates) aux étapes sensibles.

L'agent tourne dans **opencode**. Son comportement est défini par des fichiers SKILL.md (progressive disclosure), et ses capacités techniques par des serveurs MCP et des scripts Python.

## Principe de fonctionnement

L'agent lit un fichier `SKILL.md` racine à chaque interaction. Ce fichier est un **routeur** : il contient les règles toujours actives (gestion d'erreur, protocole des Gates, budget de tokens) et une table qui associe les déclencheurs utilisateur aux procédures. Quand l'utilisateur écrit « traite le ticket AER_XXX », l'agent charge la procédure correspondante, et seulement au moment voulu, il charge les sous-skills dont il a besoin. C'est le **progressive disclosure** : rien n'est en contexte tant que ce n'est pas nécessaire, ce qui économise les tokens et garde l'agent focalisé.

## Architecture des fichiers

```
SKILL.md                          Routeur + règles toujours actives
skills/
  process-jira-ticket/SKILL.md    La procédure principale (15 étapes)
  build-config/SKILL.md           Wrappers de skills : décrivent quand
  validate-specs/SKILL.md           appeler chaque tool et comment
  test-starburst/SKILL.md           interpréter son résultat
  final-validation/SKILL.md
  gates/SKILL.md                  Protocole Go/NoGo commun
  kb-lookup/SKILL.md              (héritage — le lookup est désormais
                                   automatique côté serveur)
tools/                            Scripts Python métier (exécutables seuls)
  build_config.py
  validate_specs.py
  push_sql.py                     (+ fix_sentinels.py en interne)
  test_starburst.py
  final_validation.py
mcp_servers/
  aidpgen_tools/                  Serveur MCP unifié (4 tools + KB)
  confluence_attachments.py       Serveur MCP dédié aux binaires Confluence
opencode.json                     Config : serveurs MCP + variables d'env
.env                              Credentials Starburst (hors git)
```

## Les skills

Une skill est un fichier Markdown avec un en-tête YAML (`name`, `description`) qui décrit une capacité. L'en-tête sert à l'agent pour décider quand la charger.

La **skill racine** porte les règles non négociables : arrêt immédiat sur erreur critique, un Gate est toujours un Go/NoGo, ne jamais éditer un fichier généré, budget de sortie limité.

La **procédure `process-jira-ticket`** enchaîne les 15 étapes : vérification et passage du statut JIRA, récupération du ticket et de la page Confluence, construction du `config.yaml`, téléchargement des Excel, validation des specs, [Gate 2], génération SQL, test Starburst, validation finale, [Gate 3], passage en Done, commentaire récapitulatif JIRA.

Les **skills de tools** (build-config, validate-specs, test-starburst, final-validation) sont de fines couches documentaires : elles disent quel tool MCP appeler, avec quels arguments, et surtout comment lire le résultat structuré (que faire selon `blocking`, `severity`, la grille de propagation, etc.).

La **skill gates** définit le protocole commun : poser une question Go/NoGo, ne plus appeler aucun tool, attendre la réponse de l'utilisateur.

## Les tools et les serveurs MCP

Distinction importante : un **serveur MCP** est un processus qui expose des **tools** (fonctions typées) à l'agent. Ce n'est pas la même chose.

**`aidpgen-tools`** est le serveur principal. Il expose quatre tools métier, chacun défini dans son propre module Python pour la lisibilité :

- `build_config` — extrait le YAML de config depuis la page Confluence.
- `validate_specs` — vérifie la cohérence croisée des deux fichiers Excel (10 checks).
- `test_starburst` — exécute le SQL généré sur Starburst et produit une grille de propagation (comptages par table × couche). Ce tool est **asynchrone** (voir plus bas).
- `final_validation` — interroge Starburst sur les rejets et les stats COUNT / COUNT DISTINCT PK.

Chaque tool est un **wrapper** : il lance le script Python correspondant sous `tools/` via subprocess, capture son rapport JSON, et retourne un résultat structuré directement exploitable par l'agent (au lieu d'un stdout brut à reparser). Les scripts restent exécutables seuls en ligne de commande, ce qui facilite le debug.

**`confluence-attachments`** est un serveur séparé qui télécharge les pièces jointes binaires (les Excel) via un appel REST direct à Confluence avec un PAT. Il est isolé car le gateway MCP corporate `DevopsProdITG` ne sait pas gérer les binaires — il renvoie du HTML de login.

**`DevopsProdITG`** est le gateway MCP corporate (distant). Il gère les opérations texte : lecture/écriture JIRA, lecture/écriture Confluence, GitLab. L'agent l'utilise pour lire le ticket, récupérer la page Confluence, commenter le ticket et changer son statut.

## Mécanismes transverses

**Résolution des chemins.** Les dossiers de tickets vivent toujours sous `/home/coder/<ticket_key>`, un chemin absolu indépendant du répertoire courant. Cette convention (variable `AIDPGEN_TICKETS_BASE`) garantit que tous les tools trouvent le même dossier, quel que soit le contexte d'exécution — c'était une source majeure de bugs non déterministes.

**Post-traitement SQL.** `push_sql.py` corrige automatiquement deux défauts du générateur v2 : les colonnes VARCHAR de la couche raw, et les sentinelles `COALESCE(col, -1)` qui provoquent des erreurs de type sur Starburst (remplacées par un sentinel adapté au type de chaque colonne).

**Gestion d'erreur automatisée.** Quand un tool rencontre une erreur critique (connexion, credentials), il fait lui-même le lookup dans la base de connaissances Confluence et renvoie le lien de la page pertinente directement dans sa réponse (champ `kb_block`). L'agent n'a plus qu'à relayer ce bloc — il ne décide rien. En parallèle, l'agent poste un commentaire d'incident sur le ticket JIRA et passe son statut en « Waiting for info ».

**Tool asynchrone.** `test_starburst` peut durer des dizaines de minutes sur de gros pipelines. Il fonctionne donc en mode job : `test_starburst_start` rend un `job_id` immédiatement, puis l'agent interroge `test_starburst_status` par petits appels courts pour suivre la progression et récupérer le résultat. Ça évite tout timeout MCP. Le résultat retourné est borné (seules les tables problématiques sont détaillées) pour ne pas saturer le contexte de l'agent.

**Détection multi-run.** Avant les INSERT, `test_starburst` compte les lignes déjà présentes dans les tables cibles. Si certaines ne sont pas vides, il le signale (`multi_run_detected`) pour éviter des comptages faussés par des données résiduelles d'un run précédent. L'option `truncate_before_insert` permet de repartir d'un état propre.

## Base de connaissances

La KB est une page Confluence parente dont chaque sous-page décrit une erreur type (symptômes, cause, résolution). Le serveur `aidpgen-tools` interroge cette page automatiquement lors d'une erreur critique, score les sous-pages par correspondance avec le message d'erreur, et renvoie le lien de la plus pertinente. Les utilisateurs enrichissent la KB en ajoutant des sous-pages, sans toucher au code.

## Points d'attention pour une reprise

Les serveurs MCP locaux ont besoin des bonnes variables d'environnement dans `opencode.json` (chemins absolus pour `AIDPGEN_TICKETS_BASE`, `STARBURST_ENV_FILE`, `PYTHONPATH`, et les credentials Confluence pour le lookup KB). Un serveur qui ne démarre pas apparaît en `-32000 connection closed` — vérifier alors le cwd, le PYTHONPATH et les dépendances de l'interpréteur.

Les libellés de statuts JIRA (« Waiting for info », « Done ») et les noms exacts des tools de `DevopsProdITG` dépendent de l'instance corporate — à vérifier si le workflow change.

---

Veux-tu que j'en fasse un fichier Markdown formaté prêt à déposer dans le repo (par exemple `docs/ARCHITECTURE.md`), ou cette version en clair te suffit pour l'intégrer où tu veux ?
