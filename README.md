# Retro-Reflect: A Self-Reflecting LLM Agent for Green Chemistry Retrosynthesis

This repository contains the complete code for the paper:

> **Retro-Reflect: Iterative Self-Reflection for Green Solvent Optimization in LLM-Driven Retrosynthesis**

---

## Overview

Retro-Reflect is a multi-module LLM agent that proposes multi-step retrosynthetic routes for organic molecules while optimizing for green chemistry principles (ACS GCI solvent scoring). The system uses an iterative **Planner–Critic–Reflector** loop: if the proposed route scores below the green threshold (7/10), the Reflector generates targeted feedback and the Planner re-plans, up to 3 iterations.

---

## System Architecture

```
Input SMILES
     │
     ▼
┌─────────────┐
│   Planner   │  GPT-4o (T=0.3) — proposes multi-step synthesis route
└──────┬──────┘
       │  SynthesisRoute (JSON)
       ▼
┌─────────────┐
│ValidityCheck│  RDKit + rule-based — blocks chemically impossible routes
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    Critic   │  GPT-4o (T=0.0) — scores each step via ACS GCI solvent DB
└──────┬──────┘
       │  RouteEvaluation (score 0–10, per-step feedback)
       ▼
  Score ≥ 7? ──Yes──► PASS (save route)
       │
      No
       ▼
┌─────────────┐
│  Reflector  │  GPT-4o (T=0.3) — generates targeted solvent improvement hints
└──────┬──────┘
       │
  iter < 3? ──Yes──► back to Planner
       │
      No
       ▼
     FAIL (save best-scoring route)
```

---

## Repository Structure

```
retro_reflect/
├── retro_reflect/               # Core Python package
│   ├── pipeline.py              # Main orchestrator — runs one molecule through the full loop
│   └── modules/
│       ├── models.py            # Pydantic data models (SynthesisRoute, RouteEvaluation, etc.)
│       ├── schema.py            # SMILES validation via RDKit
│       ├── planner.py           # LLM Planner: generates retrosynthetic routes as structured JSON
│       ├── critic.py            # LLM Critic: ACS GCI scoring, per-step solvent evaluation
│       ├── reflector.py         # LLM Reflector: produces structured improvement feedback
│       ├── comparator.py        # Diffs two routes (ReflectionDiff), tracks solvent changes
│       ├── rule_baseline.py     # Rule-based baseline: deterministic solvent replacement from DB
│       └── validity_check.py    # Chemical validity: reagent SMILES check + incompatibility rules
│
├── data/
│   ├── solvent_db.py            # ACS GCI solvent score database (green score per solvent)
│   ├── uspto_50k_sample.csv     # 10-molecule smoke-test set
│   ├── uspto_200.csv            # 200-molecule development set
│   ├── uspto_stratified_100.csv # 100-molecule stratified subset
│   ├── uspto_stratified_2000.csv# Main benchmark: 2000 USPTO molecules (8 reaction classes)
│   ├── chembl_ood_300.csv       # OOD set 1: 300 ChEMBL drug-like molecules
│   └── patent_recent_300.csv    # OOD set 2: 300 recent patent molecules
│
├── experiments/
│   ├── run_single.py            # Run one molecule in any mode
│   ├── run_batch.py             # Bulk evaluation on a CSV dataset (full pipeline)
│   ├── run_no_reflection.py     # Batch run: no-reflection mode on USPTO-2000
│   ├── run_rule_baseline_batch.py  # Batch run: rule baseline on USPTO-2000
│   ├── run_rule_baseline_ood.py    # Batch run: rule baseline on ChEMBL OOD
│   ├── run_rule_baseline_patent.py # Batch run: rule baseline on patent OOD
│   ├── run_ablation.py          # Ablation study: full / no-reflection / rule baseline
│   └── run_plateau_second_pass.py  # Analysis: second pass on score=6 plateau molecules
│
├── configs/
│   ├── base.yaml                # Shared defaults (model, temperature, thresholds)
│   ├── batch.yaml               # Batch evaluation settings
│   └── ablation.yaml            # Ablation study molecules
│
├── Makefile                     # Convenience commands
├── pyproject.toml
├── requirements.txt
└── .env.example                 # API key template — copy to .env and fill in
```

---

## Setup

Requires **Python 3.9+**.

```bash
# 1. Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate        # Mac/Linux
# .venv\Scripts\activate         # Windows

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure API key
cp .env.example .env
# Edit .env and set OPENAI_API_KEY=sk-...

# 4. Smoke-test (rule-based, no API key needed)
python experiments/run_single.py --smiles "CC(=O)Oc1ccccc1C(=O)O" --rule-baseline
```

### Key dependencies

| Package | Purpose |
|---|---|
| `openai >= 1.0` | LLM API calls (GPT-4o) |
| `rdkit >= 2023.03` | SMILES parsing and validation |
| `pydantic >= 2.0` | Typed data models |
| `python-dotenv` | `.env` file loading |
| `pyyaml` | Config file parsing |
| `pandas` | CSV I/O for batch experiments |

---

## Quick Start

```bash
# Full pipeline — single molecule (Aspirin)
make single SMILES="CC(=O)Oc1ccccc1C(=O)O"

# No-reflection baseline
python experiments/run_single.py --smiles "CC(=O)Oc1ccccc1C(=O)O" --no-reflection

# Rule-based baseline (no LLM, no API key needed)
python experiments/run_single.py --smiles "CC(=O)Oc1ccccc1C(=O)O" --rule-baseline
```

---

## Reproducing Experiments

All batch scripts write results to `outputs/` (created automatically).

### Main benchmark (USPTO-2000)

```bash
# Step 1: no-reflection pass (generates Planner routes + log)
python experiments/run_no_reflection.py

# Step 2: rule baseline (parses routes from Step 1 log — zero LLM calls)
python experiments/run_rule_baseline_batch.py

# Step 3: full pipeline
python experiments/run_batch.py --input data/uspto_stratified_2000.csv \
    --output outputs/final_2000_full_pipeline.csv

# Step 4: plateau analysis (score=6 molecules from Step 3)
python experiments/run_plateau_second_pass.py \
    --csv-in outputs/final_2000_full_pipeline.csv \
    --log-in outputs/final_2000_full_pipeline_log.txt
```

### OOD benchmarks (ChEMBL-300 / Patent-300)

```bash
# ChEMBL OOD
python experiments/run_batch.py --input data/chembl_ood_300.csv \
    --output outputs/ood_chembl_300_full_pipeline.csv
python experiments/run_rule_baseline_ood.py

# Patent OOD
python experiments/run_batch.py --input data/patent_recent_300.csv \
    --output outputs/ood_patent_300_full_pipeline.csv
python experiments/run_rule_baseline_patent.py
```

### Ablation study

```bash
# Single molecule
python experiments/run_ablation.py --smiles "CC(=O)Oc1ccccc1C(=O)O"

# All 5 molecules from configs/ablation.yaml
make ablation-all
```

---

## Output CSV Format

All batch output CSVs share the same schema:

| Column | Type | Description |
|---|---|---|
| `run_id` | str | Unique 12-char hex ID per run |
| `name` | str | Molecule name or dataset ID |
| `smiles` | str | Input SMILES string |
| `passed` | bool | True if final green score ≥ 7/10 |
| `final_green_score` | float | ACS GCI composite score (0–10) |
| `pmi` | float | Process Mass Intensity (lower = greener) |
| `total_iterations` | int | Number of Planner iterations used (1–3) |

---

## Configuration Reference

`configs/base.yaml` controls all pipeline parameters:

```yaml
llm:
  planner_model: "gpt-4o"
  planner_temperature: 0.3   # Higher = more creative routes
  critic_model: "claude-sonnet-4-5"
  critic_temperature: 0.0    # Deterministic scoring

pipeline:
  max_iterations: 3          # Maximum Planner–Critic–Reflector loops
  green_score_threshold: 7   # Pass threshold out of 10 (ACS GCI)
  max_validity_retries: 2    # Max re-plans triggered by ValidityCheck
```

Parameters can also be set via environment variables in `.env` (see `.env.example`).
