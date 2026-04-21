# Seed-Biased RBP4 Ligand Generation

End-to-end RBP4 ligand discovery notebook combining:
- graph neural network (GATv2) scoring,
- medicinal-chemistry analog generation,
- hosted AutoDock Vina docking,
- parent identity/bioactivity enrichment,
- final analog-level triage with retinol benchmarking.

## Repository Contents

- `RBP4 AI Ligand Generator.ipynb`: main pipeline notebook
- `RBP4_Model_Workflow_and_Reproducibility.md`: deep technical workflow document
- `requirements.txt`: Python dependencies
- `.gitignore`: safe defaults for notebooks, outputs, caches, models
- `INSTALL.md`: environment setup on Windows/macOS/Linux
- `RUNBOOK.md`: exact execution order (run-all and staged)
- `PROJECT_STRUCTURE.md`: expected folder/file layout
- `DATA_POLICY.md`: what to keep private and what not to upload
- `REPRODUCIBILITY_CHECKLIST.md`: handoff/reproduction checklist

## Quick Start

1. Create and activate a Python environment (see `INSTALL.md`).
2. Install dependencies from `requirements.txt`.
3. Open `RBP4 AI Ligand Generator.ipynb`.
4. Set run flag near the top:
   - `RUN_ITERATIVE_ANALOG_LOOP = False` for default stable full run
   - `True` to include iterative analog expansion
5. Run notebook top-to-bottom (see `RUNBOOK.md`).

## Key Outputs

- `outputs/enamine1123_scored.csv`
- `outputs/enamine1123_top5.csv`
- `outputs/enamine1123_top5_analogs.csv`
- `outputs/docking/docking_best_pose_per_ligand.csv`
- `outputs/enamine1123_final_compounds_enriched.csv`
- `outputs/final_candidates_tiered.csv`
- `outputs/final_candidates_tiered_ensemble.csv`
- optional iterative outputs under `outputs/iterative_analog_docking/`

## Important Notes

- Docking uses hosted endpoint configured in the notebook (`/api/docking`).
- Retinol is docked as an explicit benchmark (`retinol_reference` batch).
- This is a triage workflow, not proof of binding/efficacy.
- Review `DATA_POLICY.md` before making the repository public.

## Citation / Scientific Context

The workflow follows a practical "model sensitivity + chemistry-aware triage" philosophy inspired by:
- Kang et al., JCIM 2022, DOI: `10.1021/acs.jcim.1c01545`

