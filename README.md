# timegated-llm-sepsis-agent

Analysis code for:

> Saxena A, Sinha RK. **A small open-weight LLM agent for early warning of ICU-onset sepsis under strict time-gating: a leakage-controlled replay study in MIMIC-III.** *Manuscript submitted.* [Add journal, year and DOI when available.]

This repository contains one Jupyter notebook that reproduces the full study, from cohort extraction to the final result tables. **It contains no MIMIC-III data, derived data or outputs.**

---

## What the code does

The study tests whether a locally served, tool-using large language model (LLM) agent can predict Sepsis-3 onset within the next 12 hours in ICU patients who were not septic during their first 6 ICU hours, without access to future information.

1. **Cohort and labels.** Adults on their first ICU stay of at least 24 h. Hourly six-component SOFA scores are computed across the CareVue and MetaVision systems. Sepsis-3 onset is suspected infection (antibiotic plus culture) plus SOFA ≥ 2.
2. **Replay environment.** Decision points are placed every 4 h. At each point the agent is bound to one stay and one timestamp. Its tools (vital signs, laboratory values, urine output, clinical notes lagged by 6 h) return only data charted and stored by that time. An automated test checks that no future timestamp is ever returned. Antibiotics, cultures and SOFA are withheld from the agent.
3. **Agent.** Qwen3.6-35B-A3B (NVFP4) served locally through vLLM, in a ReAct-style tool-calling loop of at most 8 steps. It returns JSON with a risk, an alert flag, cited evidence and missing data.
4. **Comparators.** SOFA, change in SOFA, SIRS, qSOFA, NEWS2, and gradient boosting (three feature sets, tuned and Platt-calibrated).
5. **Evaluation.**
   - AUROC and AUPRC with stay-level bootstrap confidence intervals.
   - Inverse-probability weighting to the true prevalence.
   - Calibration, alert burden and decision curves.
   - Stacking, and numeric faithfulness of the agent's evidence.
   - Paired ablations, repeated runs, trend and hybrid agents.
   - An independent replication sample and full trajectories.
   - Label-variant, temporal and leave-one-unit-out validation.

## Repository contents

| File | Description |
|---|---|
| `sepsis_agent_mimic3_code_availability.ipynb` | The complete analysis in 18 code cells, one per study part, with comments. Outputs are cleared. |
| `README.md` | This file |
| `requirements.txt` | Python packages |
| `LICENSE` | MIT license (code only) |
| `.gitignore` | Keeps data and outputs out of the repository |

## Data access (required)

MIMIC-III v1.4 is a restricted-access database. To run this code you must:

1. Complete the required training and become a credentialed user on [PhysioNet](https://physionet.org/).
2. Sign the data use agreement for [MIMIC-III v1.4](https://physionet.org/content/mimiciii/1.4/).
3. Load the MIMIC-III tables into a local DuckDB file, using the standard lower-case table names (`patients`, `admissions`, `icustays`, `chartevents`, `labevents`, `noteevents`, `prescriptions`, `microbiologyevents`, `inputevents_mv`, `inputevents_cv`, `outputevents`, `procedureevents_mv`, …).

The notebook attaches this file **read-only** and writes all derived tables to a separate work database.

> **Data use agreement.** Files the notebook writes to `OUT_DIR` (agent traces, evaluation samples, feature tables, intermediate results) contain patient-level information. Do not commit them to this or any public repository, and share them only with credentialed PhysioNet users. All LLM inference runs on a local server; no patient data are sent to external services.

## Requirements

**Software**
- Python 3.12
- `duckdb`, `pandas`, `numpy`, `scikit-learn`, `joblib`, `pyarrow`, `matplotlib`
- `openai` (Python client only, pointed at the local vLLM server)
- `openpyxl` (optional, for the results workbook)

```bash
pip install -r requirements.txt
```

**LLM server** (needed only for Parts 7–8, 11 and 14)
- vLLM with tool calling enabled. The study used version 0.21.0 (development build, commit 2325b6f0).
- Model: `nvidia/Qwen3.6-35B-A3B-NVFP4`, served at an OpenAI-compatible endpoint (default `http://localhost:8001/v1`).

**Hardware used**
- NVIDIA DGX Spark (128 GB unified memory).
- The DuckDB memory limit was set to 28 GB so that it could share memory with vLLM.

## Configuration

Edit the top of **Part 0** or set an environment variable:

| Setting | Default | Meaning |
|---|---|---|
| `MIMIC_DB` (env) / `DB_PATH` | `~/mimic_project/mimic3.duckdb` | MIMIC-III DuckDB file (read-only) |
| `OUT_DIR` | `~/mimic_project/agent_runs` | Output folder (patient-level, keep private) |
| `VLLM_URL`, `MODEL` | `http://localhost:8001/v1`, `nvidia/Qwen3.6-35B-A3B-NVFP4` | Local LLM endpoint |
| `MIN_ONSET_HR`, `DECISION_EVERY_H`, `HORIZON_H` | 6, 4, 12 | Study design |
| `HIDE_INFECTION_DATA`, `ENABLE_SOFA_TOOL`, `THINKING` | True, False, True | Main configuration (changed only in ablations) |
| `SEED` | 42 | Global random seed |

## How to run

Run the cells from top to bottom. LLM runs are **resumable**: results are appended to JSONL files, and re-running a cell continues where it stopped.

| Part | Content | LLM needed | Approx. time* |
|---|---|---|---|
| 0 | Configuration, helpers, database connection | No | seconds |
| 1–5 | Cohort, suspected infection, hourly SOFA, Sepsis-3 labels, decision points and evaluation sample | No | 30–60 min |
| 6 | Time-gated tools and leakage test | No | minutes |
| 7–8 | Agent loop and main run (568 decision points) | Yes | ~2 h |
| 9 | Evaluation: discrimination, alerts, bootstrap CIs, faithfulness | No | minutes |
| 11 | Five paired ablations | Yes | ~8 h |
| 12 | Extended evaluation: features, classic scores, tuned GB, weighting, calibration, decision curves, stacking, onset drivers | No | 1–2 h |
| 13 | Label sensitivity analyses | No | ~30 min |
| 14 | Repeated, trends and hybrid agents; replication sample; trajectories | Yes | ~15 h |
| 15 | Temporal, leave-one-unit-out and subgroup validation | No | ~1 h |
| 16 | Collect all result tables into one workbook | No | seconds |

\*At a concurrency of 8 on the hardware above. All agent runs together made about 8,300 LLM calls.

Part 10 of the original analysis held planning notes only and has no code.

## Reproducibility notes

- **Sampling.** Thinking mode used temperature 0.6, top-p 0.95 and top-k 20. Non-thinking mode used 0.7, 0.8 and 20. The seed for each step is 42 + step, and the maximum is 8,192 tokens per step.
- **Same points everywhere.** The evaluation sample is frozen in `eval_points.csv` on the first run, so every method and ablation is scored on identical points.
- **Run-to-run variation.** LLM outputs vary between runs even with fixed seeds. The study quantifies this with repeated runs (Part 14).
- **Leakage test.** Run it before any new tool is used.

## Citation

If you use this code, please cite the article above. [Add BibTeX once published.]

## License

MIT (see `LICENSE`). The license applies to the code only. MIMIC-III remains governed by the PhysioNet Credentialed Health Data License and data use agreement.

## Contact

Aditya Saxena, School of Digital Health, IIHMR University, Jaipur, India · aditya@iihmr.edu.in
