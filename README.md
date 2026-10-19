# Evaluating Qualitative Judgment in LLM-Generated Credit Ratings

Reproducibility package for the study *"Evaluating Qualitative Judgment in
LLM-Generated Credit Ratings: A Validation Framework for LLM-as-Judge Without
Expert Ground Truth."*

> **Anonymized for double-blind review.** Authors, affiliations, funding, and venue
> are omitted intentionally. Please do not add identifying information to this
> repository or its commit history.

## What this is

The study asks how to evaluate the *qualitative judgment* in an LLM-generated credit
rating when expert ground truth is unavailable. Technical, reference-free metrics
(e.g. RAGAS-style faithfulness) score whether an output is consistent with its
inputs, not whether the resulting grade is defensible. We instead certify an LLM
judge *indirectly*, using only public data, through convergent probes:

1. **Perturbation sensitivity** — a sound judge penalizes outputs whose qualitative
   reasoning is degraded (inconsistency, missing or reversed justification), while a
   purely technical metric does not.
2. **Known-anchor recovery** — the judge passes unambiguous, public-rating anchors.
3. **Inter-judge agreement / self-consistency** — across models, prompts and order.

A quantitative-only baseline (a **SACP proxy**) with a predictive interval defines
each firm's standalone rank; the signed **notch gap Δ** between the agency rating and
that proxy marks the **qualitative-override** cases that stress-test contextual
judgment. A faithfulness metric grounded in the **fixed quantitative input** (not the
rationale) is run on the same perturbations as a contrast control.

## Key idea: the perturbation contrast

Each generated `(rating, rationale)` is corrupted in five controlled ways and scored
by both an LLM judge (rubric, notch-gap aware) and the technical faithfulness metric:

| Code | Perturbation | What it probes | Expected: a sound judge … |
| --- | --- | --- | --- |
| P1 | rating–rationale inconsistency | conclusion coherence | drops; metric flat |
| P2 | remove the qualitative justification | qualitative judgment | drops; metric flat |
| P3 | inject a factual (quantitative) error | factual faithfulness | both drop (metric's domain) |
| P4 | reverse the support direction | reasoning validity | drops hardest; metric partial |
| P5 | verbosity only (content unchanged) | style robustness | flat (bias control) |

## Repository layout

```
credit-rating-llm-judge/
├── data/
│   ├── raw/          # public datasets (not committed — see data/README.md)
│   └── processed/    # generated artifacts (git-ignored)
├── notebooks/        # 00–06, one self-contained code cell each, run in order
├── results/
│   ├── figures/      # PNG + PDF, grayscale, 600 dpi
│   └── tables/       # CSV + LaTeX
├── requirements.txt
├── .env.example      # copy to .env; never commit the real key
└── README.md
```

## Notebooks

| Notebook | Purpose |
| --- | --- |
| `00_setup` | Environment check; plotting/table conventions; self-test figure |
| `01_data_construction` | Merge public financials + agency ratings into the corpus |
| `02_sacp_proxy_and_override` | SACP proxy with an 80% predictive interval; notch gap Δ; override subset |
| `03_generation` | LLM generates a gap-matched rating + rationale (multi-model) |
| `04_perturbation` | Build P1–P5 perturbed variants |
| `05_judge_evaluation` | LLM-as-judge rubric scoring + input-grounded faithfulness |
| `06_analysis_and_figures` | Dose-response, per-perturbation contrast, significance tests |

Each notebook is **self-contained in a single code cell** with English comments and a
header naming the notebook. Missing inputs are detected and the notebook exits
cleanly, so notebooks can be run or re-run in any order.

## Conventions

- **Figures:** seaborn, grayscale, no captions, saved as **PNG and PDF at 600 dpi**
  via the shared `save_fig()` helper.
- **Tables:** CSV and LaTeX via `save_table()`.
- **Seed:** `SEED = 42` throughout.
- **Storage:** Parquet when an engine is installed, otherwise CSV — no code change.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate   # optional
pip install -r requirements.txt
cp .env.example .env            # then fill in OPENAI_API_KEY and model names
```

Acquire the raw data per [`data/README.md`](data/README.md). Then run the notebooks
`00` → `06`, or headless:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/0*.ipynb
```

The LLM notebooks (`03`, `05`) skip automatically when no API key is present.

## Pilot results

A first end-to-end run on the public Kaggle corporate-rating corpus (2,029 US
issuers; SACP proxy with an 80% predictive interval → 505 override firms, 24.9%)
generated ratings with three models and scored six perturbation families (150
outputs each). Judge = `qualitative_adequacy` (1–5); metric = faithfulness (×5).
Significance by Wilcoxon signed-rank, paired by `(case_id, model_tag)`, n = 150;
the divergence family is Holm-corrected.

| Perturbation | Judge drop (dz) | Metric drop (×5) | Divergence | Holm | Reading |
| --- | --- | --- | --- | --- | --- |
| P1 inconsistency | 0.51 (−0.68) | 0.06 (ns) | +0.45 | *** | judge more sensitive |
| P2 remove qualitative | 2.88 (−2.68) | −0.23 (rises) | +3.11 | *** | judge more sensitive (max) |
| P3 factual error | 0.47 (−0.81) | 1.09 | −0.61 | *** | metric more sensitive (complementary) |
| P4 direction flip | 2.95 (−2.84) | 1.41 | +1.54 | *** | judge more sensitive |
| P5 verbosity | 0.89 (−1.28) | 0.93 | −0.05 | ns | no difference |

Three findings: (1) on qualitative/judgment defects (P1, P2, P4) the judge drops
significantly more than the technical metric — for P2 the metric even rises; (2) on a
quantitative error (P3) the metric drops more, evidence of complementarity rather than
judge superiority; (3) verbosity (P5) moves both alike, so the judge's verbosity
sensitivity is not significantly greater than the metric's.

These figures are a pilot (first-pass case cap); the full run re-reports case count,
model configuration and judge configuration.

## LLM configuration

Model names are read from `.env` (`LLM_MODEL_WEAK`, `LLM_MODEL_MID`,
`LLM_MODEL_STRONG`) so the model tier can be swapped without touching code.

## License

Code is released for review; a permissive license will be attached upon
de-anonymization. Third-party datasets retain their original licenses.
