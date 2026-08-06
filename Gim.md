Bien cadré. Voici les deux fonctionnalités.

Point clé de répartition : le commentaire JIRA passe par `DevopsProdITG` (donc c'est de l'orchestration côté agent, ça vit dans la procédure), tandis que la vérification du contenu des tables touche Starburst (donc c'est côté tool MCP, dans `test_starburst`). Chacune va à sa bonne couche.

## Fonctionnalité 1 — Commentaire JIRA récapitulatif

C'est de l'orchestration : l'agent, à la fin de la procédure, écrit un commentaire selon l'issue. Ça se pilote entièrement dans `skills/process-jira-ticket/SKILL.md`. Aucun code Python.

### Modification de la procédure

Deux points de sortie à gérer : la sortie en échec (n'importe quel CRITICAL) et la sortie en succès (fin normale).

**Pour la sortie en échec** — modifie le CRITICAL failure handling dans le top-level `SKILL.md`. Actuellement il rapporte l'erreur et attend. Ajoute l'écriture JIRA juste avant d'attendre :

```markdown
## CRITICAL — Failure handling (always active)

When a tool result has `status == "CRITICAL_FAILURE"` or `blocking == true`:

1. STOP the procedure immediately.
2. Post a JIRA incident comment via DevopsProdITG add-comment on the ticket:

   ```
   ❌ Traitement AiDPGen interrompu

   Étape : <nom de l'étape en échec>
   Erreur :
   <le champ error du tool, tronqué à ~500 caractères>

   <le champ kb_block du tool, tel quel s'il est présent>

   Statut passé en "Waiting for info" — intervention requise avant relance.
   ```

3. Transition the JIRA ticket to "Waiting for info" via DevopsProdITG.
4. Report the same information to the user, including the kb_block.
5. Wait for the user's reply — do NOT call any tool while waiting.

If the JIRA comment or transition itself fails, report that to the user but
do NOT loop — a JIRA write failure must not mask the original error.
```

**Pour la sortie en succès** — remplace ton Step "Final summary" pour qu'il écrive aussi un commentaire concis avant le récap à l'utilisateur :

```markdown
### Step 15 — Final summary + JIRA report

First, post a concise success comment via DevopsProdITG add-comment:

```
✅ Traitement AiDPGen terminé

Tables traitées : <nombre de tables du propagation grid>
Santé pipeline  : <résumé by_health, ex: "38 consistent, 2 varying">
Rejets          : <rejects.total du final_validation, ou "aucun">
Alertes         : <alerts.total, ou "aucune">

Rapports :
- Test Starburst : <report_path>
- Validation     : <report_path final_validation>
```

The ticket status is already "Done" (set at Step 13). Do NOT change it here.

Then post the user-facing summary (unchanged from before).
```

### Un garde-fou important

Le statut "Waiting for info" et le statut "Done" sont mutuellement exclusifs. La procédure met "Done" au Step 13 **seulement** si tout a réussi jusque-là. Si un CRITICAL survient avant, on n'atteint jamais le Step 13, et c'est le handler qui met "Waiting for info". Les deux chemins ne se croisent pas — pas de risque de double transition contradictoire.

Vérifie juste que ton workflow JIRA autorise la transition vers "Waiting for info" depuis "In Progress". Si le nom exact du statut diffère dans ton instance (par exemple "Waiting for support" ou un statut custom), ajuste le libellé dans le handler.

## Fonctionnalité 2 — Vérification du contenu des tables avant insert

Ça touche Starburst, donc ça vit dans `test_starburst.py` (le script) et remonte via le tool MCP. Le principe : avant la phase d'INSERT, on compte les lignes déjà présentes dans les tables cibles. Si certaines ne sont pas vides, on le signale.

Ça se combine avec le `--truncate-before-insert` qu'on avait ajouté : soit l'utilisateur veut écraser (truncate), soit il veut être prévenu. On fait les deux via un mode.

### Modification de `tools/test_starburst.py`

Ajoute une option de pré-check dans le parser :

```python
parser.add_argument(
    "--precheck-content", action="store_true",
    help="Before running INSERTs, count existing rows in each target table "
         "and report which are already populated (multi-run detection).",
)
```

Puis, dans la phase 3, juste avant la boucle d'INSERT d'une couche (et avant le truncate s'il est actif), insère le pré-check :

```python
    # --- NEW: pre-insert content check (multi-run detection) ---
    prefilled = []  # list of (layer, table, existing_count)
    if args.precheck_content:
        print("\n=== Phase 3.0 - PRE-INSERT CONTENT CHECK ===")
        for layer in LAYER_ORDER:
            if layer not in layers:
                continue
            layer_targets = targets_by_layer.get(layer, [])
            # raw is backed by the bucket; its "content" is the CSVs, skip it
            if layer == "raw" or not layer_targets:
                continue
            for tgt in layer_targets:
                start = time.time()
                try:
                    cur = conn.cursor()
                    cur.execute(f"SELECT COUNT(*) FROM {tgt}")
                    rows = cur.fetchall()
                    existing = rows[0][0] if rows else 0
                except Exception:
                    existing = None  # table may not exist yet — that's fine
                dur = time.time() - start
                if existing and existing > 0:
                    prefilled.append({"layer": layer, "table": tgt,
                                      "existing_rows": existing})
                    print(f"  [WARN] {tgt} already has {existing:,} row(s)")
        if not prefilled:
            print("  All target tables empty or absent — clean run.")
        report.append({"phase": "precheck", "prefilled": prefilled})
```

Le `report.append` avec `phase: "precheck"` fait remonter l'info dans le JSON, donc le tool MCP peut la lire.

### Modification du tool MCP `test_starburst.py`

Passe l'option et remonte le résultat. Dans `test_starburst_start`, ajoute le flag :

```python
def test_starburst_start(
    ticket_key: str,
    generated_dir: str | None = None,
    env_file: str | None = None,
    report_out: str | None = None,
    layers: str = ",".join(LAYER_ORDER),
    truncate_before_insert: bool = False,
    precheck_content: bool = True,
) -> dict[str, Any]:
    ...
    args = [ ... ]
    if truncate_before_insert:
        args.append("--truncate-before-insert")
    if precheck_content:
        args.append("--precheck-content")
    ...
```

J'ai mis `truncate_before_insert` à `False` par défaut et `precheck_content` à `True` : le comportement par défaut devient "je préviens, je n'écrase pas". C'est cohérent avec ton choix "prévenir l'utilisateur".

Puis, dans `test_starburst_status`, extrais le pré-check du rapport pour le mettre en top-level :

```python
    # Extract precheck info if present
    prefilled = []
    for entry in data:
        if entry.get("phase") == "precheck":
            prefilled = entry.get("prefilled", [])
            break

    result: dict[str, Any] = {
        "status": "done",
        ...
        "prefilled_tables": prefilled,
        "multi_run_detected": len(prefilled) > 0,
    }
```

### Modification de la skill test-starburst

Ajoute une section pour que l'agent réagisse au multi-run :

```markdown
## Multi-run detection

The result includes:
- `multi_run_detected` (bool) — True if some target tables already had rows
  before the INSERTs ran.
- `prefilled_tables` — list of {layer, table, existing_rows}.

If `multi_run_detected == true`, warn the user clearly BEFORE presenting the
propagation grid:

```
⚠ Multi-run détecté : certaines tables contenaient déjà des données avant
ce run. Les comptages ci-dessous peuvent inclure d'anciennes lignes.

Tables concernées :
<une ligne par table : nom + nombre de lignes préexistantes>

Options :
- Relancer avec truncate_before_insert=true pour repartir propre.
- Ou vérifier manuellement si le cumul est intentionnel.
```

Do NOT silently proceed as if the counts are clean when multi_run_detected
is true — the user must know the numbers may be inflated.
```

## Comment les deux fonctionnalités se combinent

Prends un scénario réel de multi-run qui tourne mal :

L'utilisateur relance un ticket déjà partiellement traité. Le pré-check détecte que 12 tables ont déjà du contenu. Le tool remonte `multi_run_detected: true`. L'agent prévient l'utilisateur avant le Gate 3. Si l'utilisateur décide quand même de continuer et qu'une erreur de connexion survient plus loin, le CRITICAL handler écrit un commentaire JIRA d'incident, passe le ticket en "Waiting for info", et attend. L'utilisateur a toutes les infos : le multi-run signalé, l'erreur commentée dans JIRA, le lien KB.

Les deux fonctionnalités se renforcent : la première trace ce qui s'est passé dans JIRA, la seconde évite les faux diagnostics dus aux données résiduelles.

## Points à vérifier de ton côté

Le libellé exact du statut JIRA "Waiting for info" — vérifie qu'il correspond à ton workflow et que la transition depuis "In Progress" est autorisée.

Le nom exact du tool add-comment de `DevopsProdITG` — demande à l'agent de lister les tools du serveur pour avoir le nom précis, et ajuste dans la skill si besoin.

Le comportement du pré-check sur `raw` — je l'ai volontairement exclu, parce que raw pointe sur le bucket et "avoir du contenu" y est normal. Si ta couche raw est une vraie table matérialisée et non une external table, dis-le moi et j'ajuste.

Tu veux que je te livre les fichiers complets modifiés (`test_starburst.py` script + tool MCP + skill), ou ces patches ciblés te suffisent pour intégrer ?
