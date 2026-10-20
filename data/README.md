# Data acquisition

Raw datasets are **not redistributed** in this repository (licensing). Download them
into `data/raw/` as described below, then run the notebooks in order. Agency ratings
are facts; narrative text and third-party compilations retain their own licenses —
check each source before redistributing.

## Required

- **Corporate credit ratings + financial ratios** → `data/raw/corporate_rating.csv`
  Kaggle "Corporate Credit Rating with Financial Ratios" (US issuers, 2010–2016;
  S&P, Moody's, Fitch, Egan-Jones). Columns include `Rating`, sector, and ~25 ratios.

## Optional / recommended

- **SEC EDGAR XBRL financial statement data sets** → `data/raw/edgar/`
  Free bulk download (DERA) or the `data.sec.gov` Company Facts / frames JSON API.
  Used to corroborate or extend the quantitative features.
- **Annual-report text (CCRD / CreditARF)** → `data/raw/ccrd/`
  Financials paired with annual-report narrative; source of the qualitative context
  fed to the generator.

## Pipeline order

1. `01_data_construction` → `data/processed/corpus.parquet`
2. `02_sacp_proxy_and_override` → `corpus_scored.parquet`, `override_subset.parquet`
3. `03_generation` → `generations.parquet` (needs `.env`)
4. `04_perturbation` → `perturbations.parquet`
5. `05_judge_evaluation` → `judge_scores.parquet` (needs `.env`)
6. `06_analysis_and_figures` → figures/tables in `results/`

Every notebook detects missing inputs and exits with a message, so running out of
order is safe.
