# RBP4 AI Ligand Generator: Current Pipeline and Reproducibility Guide

## 1) Scope and Design Intent

This notebook is an end-to-end, laptop-practical RBP4 ligand discovery workflow that combines:

- target-aware graph neural network scoring,
- rule-based medicinal chemistry analog generation,
- hosted AutoDock Vina docking,
- parent identity and indication enrichment,
- analog-level final triage with a retinol benchmark.

It is built for **transparent hit triage**, not unconstrained de novo generation.  
The system is designed so a medicinal chemist can inspect structure-level decisions and a software/ML reader can audit the scoring and data flow.

---

## 2) High-Level Pipeline (Start to Finish)

1. Install dependencies and set runtime flags.
2. Prepare receptor from 5NU7 (including retinol pocket reference).
3. Build molecular graphs (`mol_graph.py` cell).
4. Define GNN (`gnn_model.py` cell).
5. Train/fine-tune checkpoint (`train_gnn.py` cell) or reuse `models/rbp4_gnn.pt`.
6. Score Enamine FDA library and generate structural analogs.
7. Dock top parents + analogs + explicit all-trans retinol benchmark.
8. Enrich parent names/CIDs from PubChem and render post-docking figure.
9. Produce English summary (bioactivity + ocular relevance + analog transforms).
10. Generate final analog triage tables (single-model and ensemble variants).
11. Optionally run iterative analog expansion+docking loop.

The notebook now supports safe `Run All` with optional heavy blocks controlled by flags.

---

## 3) Inputs, Outputs, and Core Artifacts

### Primary inputs

- Notebook: `RBP4 AI Ligand Generator.ipynb`
- Screening library: `data/Enamine_FDA_approved_Drugs_plated_1123cmpds_20250601.smiles`
- Known positives (few-shot anchors): `KNOWN_BINDERS` + positive control definitions in notebook
- Receptor/Pocket files:
  - `data/5NU7_raw.pdb`
  - `data/5NU7_protein.pdb`
  - `data/5NU7_retinol.pdb`
  - `data/5NU7_receptor.pdbqt`
- Checkpoint: `models/rbp4_gnn.pt`

### Primary outputs

- Library scoring: `outputs/enamine1123_scored.csv`
- Top parents: `outputs/enamine1123_top5.csv`
- Top-parent analogs: `outputs/enamine1123_top5_analogs.csv`
- Docking summary: `outputs/docking/docking_best_pose_per_ligand.csv`
- Enriched analog table: `outputs/enamine1123_final_compounds_enriched.csv`
- Final triage:
  - `outputs/final_candidates_tiered.csv`
  - `outputs/final_candidates_tiered_ensemble.csv`
- Optional iterative loop outputs:
  - `outputs/iterative_analog_docking/iterative_all_docked.csv`
  - `outputs/iterative_analog_docking/iterative_best_final.csv`

---

## 4) Receptor and Docking Context

The receptor-prep cell:

1. downloads `5NU7` from RCSB,
2. isolates `RTL` coordinates (retinol in crystal),
3. writes protein-only structure,
4. prepares receptor PDBQT via Meeko (`--read_pdb` path for Windows robustness),
5. computes docking center from retinol centroid.

Docking uses hosted API:

- base: `https://toolbox.spartaninventory.dev`
- endpoint: `/api/docking`
- receptor: `data/5NU7_receptor.pdbqt`
- center: retinol-derived coordinates
- box: approximately `20 x 20 x 20` A
- default poses: `5`

Docked batches include:

- `top5_fda` (parents),
- `top5_analogs`,
- `retinol_reference` (explicit all-trans retinol benchmark).

This benchmark is propagated into triage as:

- `retinol_affinity`
- `delta_vs_retinol_kcal_mol` (candidate affinity minus retinol affinity)

---

## 5) Molecular Representation and Model Logic

### Graph featurization (`mol_graph.py` cell)

Each molecule is converted to PyTorch Geometric `Data` with:

- atom features (`x`): standard atom descriptors plus RBP4-biased pharmacophore flags,
- bond topology (`edge_index`) with bond attributes (`edge_attr`),
- optional labels (`y`) for supervision.

Invalid molecules are dropped safely.

### GNN architecture (`RBP4GATv2`)

The current model is a widened edge-aware GATv2 classifier:

- stacked GATv2 blocks + normalization/activation,
- global graph pooling,
- MLP readout to a single logit.

Probability is `sigmoid(logit)`.  
The notebook uses this probability for ranking and can also emit embeddings for analysis.

---

## 6) Training and Evaluation Strategy

### Training pass

- few-shot positives (`KNOWN_BINDERS` + control),
- decoys sampled from the Enamine library,
- optional positive augmentation through randomized SMILES,
- weighted BCE loss (`BCEWithLogitsLoss` with class weighting),
- Adam-family optimization + scheduler + clipping.

### Evaluation/ensemble block (optional but recommended for robustness)

- repeated stratified CV metrics:
  - AUROC
  - PR-AUC
  - Brier
  - F1
- calibration plot output
- multi-seed ensemble training
- ensemble scoring with uncertainty-aware rank:

`ensemble_rank_score = ensemble_mean - lambda * ensemble_std`

This gives a practical uncertainty penalty without heavy Bayesian machinery.

---

## 7) Ranking Logic (Chemistry + ML)

For each screened molecule, the inference cell computes model score(s) and ranking adjustments.

### Scoring

- Multi-SMILES scoring (`SCORE_SMILES_SAMPLES`) yields:
  - `score` (mean probability)
  - `score_std` (sensitivity to SMILES view)

### Structure-aware ranking signals

- retinoid similarity (`tanimoto_vs_retinol`)
- hydrocarbon pattern statistics and penalties
- optional COX de-prioritization hooks (configurable)
- CLogP and heavy-atom-size similarity to known binders
- small-size penalty
- ocular-term preference bonus (lightweight name/label-based preference)
- hard alkane-chain exclusion logic

Top parent selection is scaffold-aware (Murcko diversity) to reduce near-duplicate collapse.

---

## 8) Analog Generation Logic

Analog generation is **reaction-rule based** (RDKit SMARTS), not simple SMILES shuffling.

### Active transform families (current)

- Esterification:
  - `methyl_ester`
  - `ethyl_ester`
  - `n_propyl_ester`
  - `isopropyl_ester`
  - `tert_butyl_ester`
- Heteroatom alkylation:
  - `o_methyl`, `o_ethyl`, `n_methyl`
- Aromatic substitutions:
  - `aryl_methyl`, `aryl_chloro`, `aryl_fluoro`
- Alkyl fluorination:
  - methyl mono/di/tri fluorination
  - ethyl mono/di/tri fluorination
  - isopropyl mono/di/tri fluorination

Products are sanitized, deduplicated, and filtered.  
Fallback logic exists to guarantee fixed analog counts when chemistry rules yield too few products.

---

## 9) Enrichment and Human-Readable Reporting

The enrichment/summary stages resolve parent identity and use indications:

- PubChem PUG REST (CID/name/synonym/title lookups),
- OpenFDA drug labels where helpful,
- Wikipedia summary fallback,
- caching to `outputs/pubchem_name_cache.csv`.

Defensive filters remove identifier-like junk text (InChI/SMILES/CAS-like strings) from prose output.  
The English summary cell reports:

- top parent identity,
- therapeutic use / what condition is treated,
- ocular relevance interpretation,
- analog transform descriptions.

---

## 10) Final Triage Outputs

### Single-model triage (`final_candidates_tiered.csv`)

- analog-level merge with docking results,
- retains all analogs with `dock_affinity <= -8.0` (good),
- labels `dock_affinity <= -9.0` as excellent,
- computes consensus score from model + docking (+ ocular bonus where configured),
- includes retinol benchmark comparison columns.

### Ensemble triage (`final_candidates_tiered_ensemble.csv`)

- same docking thresholds,
- consensus built from ensemble rank plus docking,
- includes uncertainty terms (`ensemble_mean`, `ensemble_std`) and retinol delta.

---

## 11) Optional Iterative Expansion Loop

The notebook includes an optional iterative analog-docking cell controlled by:

- `RUN_ITERATIVE_ANALOG_LOOP`

Default is off for safer `Run All`.  
When enabled:

1. starts from best docked seed analogs,
2. selects top N per round (default 10),
3. generates 2 analogs per selected parent,
4. docks them,
5. repeats for 3 rounds total.

Final reporting is **global-best across all rounds** (not only last round), with:

- parent display name,
- parent Enamine ID,
- docking score,
- iteration origin label (`original_analog`, `first_iteration`, etc.),
- transform.

The figure legend is aligned to the same metadata.

---

## 12) Run-All Safety and Operational Guardrails

Key reliability mechanisms:

- explicit dependency and order checks (`globals()` guards),
- robust file-write fallbacks for Windows lock contention,
- docking API polling + cleanup in `finally`,
- missing-data checks before each stage,
- cached web lookups to reduce repeated external calls,
- optional heavy-stage flags to keep `Run All` stable.

Practical recommendation:

- keep optional iterative loop off during routine runs,
- rerun enrichment before iterative visualization if parent names changed.

---

## 13) Reproducibility Checklist

For a reproducible handoff/archive, capture:

1. notebook version/commit snapshot,
2. library input file checksum,
3. model checkpoint file + hash,
4. environment (`pip freeze`),
5. key constants:
   - scoring sample count
   - docking thresholds
   - analog generation counts and transform list
   - penalty/bonus weights
6. docking API endpoint and box/center parameters,
7. final CSV outputs and key figures.

---

## 14) Limitations and Interpretation Notes

- few-shot supervision remains a primary bottleneck for generalization.
- docking is a ranking heuristic, not proof of binding or efficacy.
- rule-based analog generation is tractable and interpretable but not exhaustive.
- therapeutic-use enrichment depends on external data quality.
- no ADMET, synthetic accessibility, or prospective assay loop is yet enforced end-to-end.

Use output as high-quality triage candidates for medicinal chemistry and experimental follow-up, not as standalone validation.
