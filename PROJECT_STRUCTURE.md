# Project Structure

Expected layout:

```text
RBP4 AI/
├─ RBP4 AI Ligand Generator.ipynb
├─ RBP4_Model_Workflow_and_Reproducibility.md
├─ README.md
├─ INSTALL.md
├─ RUNBOOK.md
├─ PROJECT_STRUCTURE.md
├─ DATA_POLICY.md
├─ REPRODUCIBILITY_CHECKLIST.md
├─ requirements.txt
├─ .gitignore
├─ data/
│  ├─ Enamine_FDA_approved_Drugs_plated_1123cmpds_20250601.smiles
│  ├─ known_binders.smi
│  └─ (generated receptor files)
├─ models/
│  └─ rbp4_gnn.pt
└─ outputs/
   ├─ docking/
   ├─ model_eval/
   ├─ iterative_analog_docking/
   └─ *.csv / *.png
```

Notes:
- `outputs/` and `models/` are usually regenerated and should not be committed by default.
- Keep input data licensing constraints in mind before sharing.

