# Execution Runbook

## Default Recommended Run (Stable)

1. Set top flag:
   - `RUN_ITERATIVE_ANALOG_LOOP = False`
2. Run notebook from top to bottom.

This executes:
- receptor prep
- graph/model definition
- training/inference
- analog generation
- docking (parents + analogs + retinol benchmark)
- enrichment + summaries
- final triage outputs

## Full Run Including Iterative Expansion

1. Set:
   - `RUN_ITERATIVE_ANALOG_LOOP = True`
2. Run notebook top-to-bottom.

This additionally performs:
- iterative analog generation+docking loop
- global best analog table and figure across all rounds

## Output Validation Checklist

- `outputs/docking/docking_best_pose_per_ligand.csv` exists
- `retinol_reference` rows present in docking output
- `outputs/enamine1123_final_compounds_enriched.csv` exists and has `parent_display_name`
- final triage files written:
  - `outputs/final_candidates_tiered.csv`
  - `outputs/final_candidates_tiered_ensemble.csv`

## Common Failures

- Missing globals/functions:
  - Run earlier definition cells (mol graph/model/docking cell)
- API timeout:
  - Re-run docking cell
- Parent names are Enamine IDs:
  - Re-run enrichment cell before iterative output

