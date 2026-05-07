# Retro-Reflect: A Self-Reflecting LLM Agent for Green Chemistry Retrosynthesis

This repository contains the complete code and pre-computed experiment results for the paper:

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
│   ├── run_no_reflection_v2.py  # Batch run: no-reflection mode
│   ├── run_rule_baseline_batch.py  # Batch run: rule baseline on USPTO
│   ├── run_rule_baseline_ood.py    # Batch run: rule baseline on ChEMBL OOD
│   ├── run_rule_baseline_patent.py # Batch run: rule baseline on patent OOD
│   └── run_plateau_second_pass.py  # Analysis: second pass on score=6 plateau molecules
│
├── configs/
│   ├── base.yaml                # Shared defaults (model, temperature, thresholds)
│   └── batch.yaml               # Batch evaluation settings
│
├── outputs/                     # Pre-computed experiment results
│   ├── final_2000_full_pipeline.csv      # Main benchmark — full pipeline, N=2000
│   ├── final_2000_no_reflection.csv      # Main benchmark — no reflection, N=2000
│   ├── final_2000_rule_baseline.csv      # Main benchmark — rule baseline, N=2000
│   ├── final_2000_plateau_second_pass.csv# Second-pass analysis on score=6 plateau molecules
│   ├── ood_chembl_300_full_pipeline.csv     # OOD ChEMBL — full pipeline, N=300
│   ├── ood_chembl_300_no_reflection.csv     # OOD ChEMBL — no reflection, N=300
│   ├── ood_chembl_300_rule_baseline.csv     # OOD ChEMBL — rule baseline, N=300
│   ├── ood_patent_300_full_pipeline.csv     # OOD patent — full pipeline, N=300
│   ├── ood_patent_300_no_reflection.csv     # OOD patent — no reflection, N=300
│   ├── ood_patent_300_rule_baseline.csv     # OOD patent — rule baseline, N=300
│   └── data_summary.md          # Aggregated statistics and figure data
│
├── tests/
│   └── test_modules.py          # 32 unit tests (no API key required)
│
├── Makefile                     # Convenience commands
├── conftest.py                  # pytest sys.path setup
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt         # Adds pytest
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
pip install -r requirements-dev.txt

# 3. Configure API key
cp .env.example .env
# Edit .env and set OPENAI_API_KEY=sk-...

# 4. Verify installation (no API key needed)
make test
# Expected: 32 passed
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
  critic_model: "gpt-4o"
  critic_temperature: 0.0    # Deterministic scoring

pipeline:
  max_iterations: 3          # Maximum Planner–Critic–Reflector loops
  green_score_threshold: 7   # Pass threshold out of 10 (ACS GCI)
  max_validity_retries: 2    # Max re-plans triggered by ValidityCheck
```

Parameters can also be set via environment variables in `.env` (see `.env.example`).

---

## Running Tests

```bash
make test
# or: pytest tests/ -v
# Expected: 32 passed, 0 failed — no API key required
```

Unit tests cover all modules including the Planner, Critic, Reflector, ValidityCheck, Comparator, and rule baseline. LLM calls are mocked.
