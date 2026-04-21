# Reproducibility Checklist

Use this checklist before sharing or archiving results.

## Environment
- [ ] Python version recorded
- [ ] `requirements.txt` up to date
- [ ] OS and hardware noted (CPU/GPU)

## Inputs
- [ ] Screening library filename and checksum recorded
- [ ] Known binder list version recorded
- [ ] Receptor source (PDB ID/version) recorded

## Model
- [ ] Checkpoint filename/date recorded
- [ ] Training constants recorded (epochs, lr, batch size, seeds)
- [ ] Ensemble settings recorded (if used)

## Docking
- [ ] Endpoint URL recorded
- [ ] Box center/size recorded
- [ ] Exhaustiveness/poses recorded
- [ ] Retinol benchmark affinity present

## Outputs
- [ ] Scored CSVs archived
- [ ] Docking best-pose CSV archived
- [ ] Enriched final table archived
- [ ] Final triage CSVs archived
- [ ] Optional iterative output CSVs archived

## QA
- [ ] Parent display names resolved (not only Enamine IDs)
- [ ] Analog transform labels human-readable
- [ ] Good/excellent docking cutoffs match intended values

