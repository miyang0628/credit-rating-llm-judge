# Validating LLM Judges for Automated Credit Assessment

Reproducibility package for the study *"Validating Large Language Model Judges for
Automated Credit Assessment: A Computational Framework Without Expert Ground Truth."*

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
2. **Circularity ablation** — a 2×2 over generation and judging conditions that
   separates what the judge detects on its own from what the generation protocol planted.
3. **Inter-judge agreement** and **known-anchor recovery** — reliability and calibration
   bounds on the judge.

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
| `03_generation` | LLM generates rating + rationale under `gap_given` / `gap_blind` conditions |
| `04_perturbation` | Build P1–P5 perturbed variants (carries `gen_condition`) |
| `05_judge_evaluation` | LLM-as-judge rubric (long format) under `informed`/`blind`; faithfulness; secondary judge; anchor probe |
| `06_analysis_and_figures` | Dose-response, contrast, significance; circularity ablation; inter-judge; anchor recovery |

Each notebook is **self-contained in a single code cell** with English comments and a
header naming the notebook. Missing inputs are detected and the notebook exits
cleanly, so notebooks can be run or re-run in any order. Execute `03 → 04 → 05 → 06`
for the LLM stages; the ablation adds ~4–5× to API cost, so start with a small
`N_CASES` (10–15) in `03`.

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

A first end-to-end run on the public corporate-rating corpus (2,029 US issuers; SACP
proxy with an 80% predictive interval → 505 override firms, 24.9%) generated ratings
with three models and scored the perturbation families. The headline uses the primary,
**gap-informed** judge on **gap-given** generations. Judge = `qualitative_adequacy`
(1–5); metric = faithfulness (×5). Significance by Wilcoxon signed-rank, paired by
`(case_id, model_tag)`, n = 150; the divergence family is Holm-corrected.

| Perturbation | Judge drop (dz) | Metric drop (×5) | Divergence | Holm | Reading |
| --- | --- | --- | --- | --- | --- |
| P1 inconsistency | 0.23 (−0.35) | 0.15 | +0.08 | ns | no difference |
| P2 remove qualitative | 2.90 (−2.63) | −0.49 (rises) | +3.39 | *** | judge more sensitive |
| P3 factual error | 0.42 (−0.67) | 0.87 | −0.45 | *** | metric more sensitive (complementary) |
| P4 direction flip | 2.98 (−2.77) | 1.16 | +1.82 | *** | judge more sensitive |
| P5 verbosity | 0.63 (−0.98) | 1.06 | −0.43 | *** | metric more sensitive |

On reversing or removing an override's qualitative reasoning (P2, P4) the judge drops
far more than the technical metric — for P2 the metric even rises. On a quantitative
error (P3) the metric drops more, evidence of complementarity rather than judge
superiority. Verbosity (P5) perturbs the input-grounded metric more than the judge, i.e.
the judge is comparatively style-robust.

### Circularity ablation (the key robustness check)

Because a gap-aware generator may plant a gap-matched justification, the large P2/P4
effects could reflect the generation design rather than the judge. The 2×2 crosses the
generation condition (`gap_given` / `gap_blind`) with the judge condition
(`informed` / `blind`), reporting the judge's drop versus the matched P0 baseline:

| Generation | Judge | P0 base | P2 drop | P4 drop |
| --- | --- | --- | --- | --- |
| `gap_given` | `informed` (headline) | 4.13 | **2.90** | **2.98** |
| `gap_blind` | `blind` (conservative) | 3.55 | **0.11** | **1.05** |
| `gap_blind` | `informed` | 1.10 | 0.08 | 0.07 |
| `gap_given` | `blind` | 1.00 | −1.46 | 0.00 |

- **P4 (direction flip) is robust**: even when neither the generator nor the judge is
  told the gap, reversing the support direction lowers the judge by ≈ 1.05 — the judge
  penalizes invalid reasoning on its own.
- **The large P2 (removal) effect is not robust**: it falls from 2.90 to 0.11 once the
  generator is blind to the gap, so it is substantially an artifact of seeding the
  rationale. Removal detection should be read as suggestive, not decisive.
- The two off-diagonal cells cross a told-gap with an unsupporting rationale, which
  floors the P0 baseline (≈ 1) and is not interpretable.

The ablation is itself a methodological result: reference-free judge validation on
*generated* outputs must cross generation and judging conditions, or it risks certifying
the generation protocol rather than the judge.

### Reliability and calibration bounds

- **Inter-judge agreement** (secondary model, n = 103): Spearman ρ = 0.71,
  quadratic-weighted Cohen's κ = 0.75 — substantial; the rubric scoring is reproducible.
- **Known-anchor recovery** (n = 40 rating extremes, financials only): exact 12.5%,
  within-2 22.5%, MAE 5.45 notches. The judge is **not** an absolute rating predictor;
  it is a reliable *relative* evaluator of reasoning against a given anchor, which is
  exactly what the framework asks of it.

These figures are a pilot (first-pass case cap); the effect directions, the divergence
structure, and the ablation — not the point magnitudes — are the findings we stand behind.

## LLM configuration

Model names are read from `.env` (`LLM_MODEL_WEAK`, `LLM_MODEL_MID`,
`LLM_MODEL_STRONG`) so the model tier can be swapped without touching code.

## License

Code is released for review; a permissive license will be attached upon
de-anonymization. Third-party datasets retain their original licenses.
