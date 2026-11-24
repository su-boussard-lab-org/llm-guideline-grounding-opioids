# Q-PAIN++: Answer-Option Sensitivity in LLM Opioid-Prescribing Audits

Code, prompts, model outputs and analysis for the paper:

> Soumyadeep Roy, Keyi Yu, Thomas Sandholm and Tina Hernandez-Boussard.
> **Beyond Binary Audits: Answer Options Change LLM Decisions and Bias-Audit Outcomes in Opioid Prescribing.** 2026.

Q-PAIN audits LLMs for race and gender bias in pain management by asking them to choose between two opioid
supplies, *Low (1 week)* or *High (4 weeks)*. Q-PAIN++ asks whether the audit's results depend on that fixed choice.
It adds the options *Medium (2 weeks)* and *None of the above*, compares each model with itself across option sets,
and repeats the Q-PAIN bias audit under each set.

---

## What is evaluated

**Option conditions** (same 10 post-operative vignettes × 8 race–gender profiles = 80 items):

| Condition | Answer options | Output folder |
|---|---|---|
| Tier 1 (original Q-PAIN) | Low, High | `tier1_orig` |
| Tier 1-DSF | Low, Medium, High, None of the above | `tier1_dsf` |
| Tier 1-M (ablation) | Low, Medium, High | `tier1_medium_only` |
| Tier 1-N (ablation) | Low, High, None of the above | `tier1_none_only` |
| Tier 2 (clinical risk factors added) | Low, High | `tier2_orig` |
| Tier 2-DSF | Low, Medium, High, None of the above | `tier2_dsf` |

Tier 2 adds 20 risk-factor combinations (mental-health history × opioid tolerance × pre-operative pain), giving
1,600 items per condition.

**Systems:** GPT-5.4, GPT-4.1-mini, GPT-4o-mini (OpenAI API) and Llama-3.1-8B (local, via Ollama). Each model is run
as published and with guideline retrieval ("+ retr."), in which the top 10 passages from 77 perioperative
pain-management guidelines are added to the prompt.

**Passage-order check:** the Tier 1 "+ retr." items are re-run with the retrieved passages in random and reverse
order (`tier1_retr_chunk_random_order`, `tier1_retr_chunk_reverse_order`).

---

## Repository layout

```
.
├── data/
│   ├── data_post_op.csv                                 # Q-PAIN post-operative vignettes
│   ├── qpain_postop_data_with_retrieved_evidence.csv    # same vignettes + retrieved guideline passages
│   └── clinician_feasibility_check/                     # the two clinicians' annotation forms
├── guideline-RAG/
│   ├── guideline corpus/                                # guideline chunks and metadata
│   └── RAG scripts/                                     # retrieval notebook and chunking scripts from raw guidelines (use "guidelinerag" virtual environment)
├── src/                                                 # run the experiments (notebooks)
│   ├── tier1_openai.ipynb / tier1_ollama.ipynb
│   ├── tier2_openai.ipynb / tier2_ollama.ipynb
│   ├── postprocess_llama31_gpt54.ipynb
│   ├── tier1_chunk_order_openai.ipynb / tier1_chunk_order_ollama.ipynb
│   ├── ablation_analysis/                               # original Tier 1-M / Tier 1-N scripts
│   ├── tier1.zip, tier2.zip, postprocess.zip            # original scripts the notebooks are built from
│   └── data-generated/                                  # where the notebooks write new outputs
├── experiment_results/                                  # model outputs used in the paper
├── analysis_scripts/                                    # analysis notebooks, their figures and tables
│   ├── tier1_full_analysis.ipynb
│   └── tier2_full_analysis.ipynb
├── figures/                                             # paper figures (Tier 2)
└── results/qpain_bias_audit/                            # bias-audit tables and figures (Tier 1, Tier 2)
```

---

## Setup

1. **Python environment** (either one):
   ```bash
   conda env create -f dsfaudit_conda_environment.yml
   # or
   pip install -r dsfaudit_pip_requirements.txt
   ```

   For reproducing the guideline RAG implementation from scratch, go to the "guideline-RAG" folder.
    ```bash
   conda env create -f guidelinerag_conda_environment.yml
   # or
   pip install -r guidelinerag_pip_requirements.txt
   ```

2. **OpenAI models:** create a `.env` file in `src/` (or the repository root) containing
   ```
   OPENAI_API_KEY=sk-...
   ```
3. **Llama-3.1:** install [Ollama](https://ollama.com), then
   ```bash
   ollama pull llama3.1:8b
   ollama serve
   ```
   Re-running the retrieval step also needs `ollama pull qwen3-embedding:8b`.
4. **Data:** the Q-PAIN vignettes are available on [PhysioNet](https://physionet.org/content/q-pain/1.0.0/).
   The two files the notebooks need are already in `data/`.

---

## Reproducing the experiments

**Reproduce Analysis:**: Just run the Jupyter notebooks under the "analysis_scripts" folder.

**Reproduce LLM output:** The "experiment_results" folder contains the generated LLM responses for all 8 systems across Tier-1, Tier-2, Ablation Analysis and Retrieval Sensitivity to Chunk Order experiments

Run every notebook **from the `src/` folder**, top to bottom (*Run All*). Each one reads from `data/` and writes to
`src/data-generated/<output folder>/`. If a notebook stops partway, run it again: finished output files are skipped.

**Guideline RAG implementation**: Please go to the "guideline-RAG" folder. Run the qpain_guideline_retrieve_rag_periop_pain.ipynb notebook. If you want to reproduce even the chunking from raw guidelines, please run the chunking_periop_pain_paragraph_wise_ideal_size.ipynb file (for just the perioperative guidelines used in the paper). You can also do to for the all disease guideline corpus that we release as described in the Appendix, using the chunking_allguidelines_paragraph_wise_ideal_size.ipynb notebook.

| Step | Notebook | What it does | Calls |
|---|---|---|---|
| 1 | `tier1_openai.ipynb` | Tier 1, 1-DSF, 1-M, 1-N for the three GPT models, ± retrieval | 1,920 |
| 1 | `tier1_ollama.ipynb` | Same for Llama-3.1 | 640 |
| 2 | `tier2_openai.ipynb` | Tier 2 and 2-DSF for the three GPT models, ± retrieval (about 6–9 h) | 19,200 |
| 2 | `tier2_ollama.ipynb` | Same for Llama-3.1 | 6,400 |
| 3 | `postprocess_llama31_gpt54.ipynb` | Recovers the chosen supply from GPT-5.4 and Llama-3.1 replies whose format differs from the requested one | – |
| 4 | `tier1_chunk_order_openai.ipynb` / `_ollama.ipynb` | Passage-order check (needs step 1's "+ retr." Tier 1 outputs) | 480 / 160 |

Notes:
- Decoding follows Q-PAIN: temperature 0, one response per prompt, top-5 log probabilities where the model provides
  them (GPT-5.4 does not), and 150 tokens for GPT models, 1,500 for Llama-3.1. Hosted models are not fully
  deterministic at temperature 0, so a new run can differ slightly from `experiment_results/`.
- Post-processing edits files in place. Untouched copies are kept in `src/data-generated/raw_before_postprocess/`.
- The notebooks keep Q-PAIN's column names (`gpt4o_answer`, `gpt4o_dosage`, …) for every model.

**New output file names:** `results_post_op_<model>[_retr_evidence]_<condition>_ff.csv`, where `<model>` is
`gpt4o_mini`, `gpt4_1_mini`, `gpt5_4` or `llama3_1`. In `experiment_results/` the same files have short names
(`gpt41_mini.csv`, `retr_gpt5_4.csv`, …).

---

## Analysis

The analysis notebooks in `analysis_scripts/` use the outputs in `experiment_results/`:

- `tier1_full_analysis.ipynb`: effect of the expanded options on Tier 1 (share of Low decisions, decision shift
  rate, ablations, passage order) and the clinician acceptability analysis.
- `tier2_full_analysis.ipynb`: Tier 2 results and the Q-PAIN bias audit under each option set.

Figures and tables are written to `analysis_scripts/figures/`, `analysis_scripts/results/`, `figures/` and
`results/qpain_bias_audit/`.

---

## Important caveat

This repository studies how an audit's conclusions depend on its answer options. It does not establish which supply
is clinically correct. No model or option set evaluated here should be used to guide opioid prescribing.

## Contact

Tina Hernandez-Boussard, Stanford University, boussard@stanford.edu
