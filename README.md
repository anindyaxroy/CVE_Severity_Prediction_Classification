# CVE Severity Classifier at Disclosure Time

**Author:** Anindya Roy  
**Programme:** MBA in AI & Analytics — University of Amsterdam  
**Course:** Big Data Infrastructures  
**Date:** March 2026

---

## What this project does

When a CVE is first published, NVD's severity score is not there yet. Analysts review each
entry manually and that takes days or sometimes weeks. During that window, security teams
have no official triage signal.

This project builds a classifier that predicts the severity class of a new CVE — Critical,
High, Medium, or Low — the moment it is published, using only information that actually
exists at disclosure time. It does not use CVSS scores, exploitability sub-scores, or
impact sub-scores as inputs. Those fields are produced by NVD analysts at the same time
as the official score, so using them would be cheating.

The classifier reads the vulnerability description text, any CWE weakness tags present
at publication, and ecosystem and patch timing data from the GitHub Security Advisory
database. When NVD eventually publishes the official score, the prediction is retired.
The model fills a gap. It does not replace the authoritative source.

---

## Project structure

```
.
├── data/
│   ├── raw/                    # Downloaded Parquet files from NVD and GitHub APIs
│   │   ├── nvd_cves.parquet
│   │   ├── github_advisories.parquet
│   │   ├── github_unreviewed.parquet
│   │   └── github_all.parquet
│   ├── staging/                # DuckDB database files created at runtime
│   ├── features/               # Final feature matrix
│   │   └── features.parquet
├── models/                     # Saved model weights
│   └── xgb_cve_classifier.pkl
├── reports/                    # All chart outputs
│   ├── data_preparation.png
│   ├── feature_correlation.png
│   ├── model_comparison.png
│   ├── classification_performance.png
│   ├── roc_pr_curves.png
│   ├── pr_curves.png
│   ├── baseline_comparison.png
│   ├── shap_importance.png
│   ├── tfidf_analysis.png
│   ├── vulnerability_trends.png
│   ├── ecosystem_risk_heatmap.png
│   └── reproducibility.json
├── CVE_RiskClassification_v4_Final.ipynb   # Main notebook
├── README.md
├── PREREQUISITES.md
└── .gitignore
```

---

## Data sources

**NVD REST API v2**  
The National Vulnerability Database. 341,160 CVE records pulled without date filters.
The full pull takes around 2.5 hours without an API key, or 15 minutes with one.
Register for a free key at https://nvd.nist.gov/developers/request-an-api-key

**GitHub Security Advisory API (GraphQL + REST)**  
27,945 reviewed advisories via GraphQL. An additional pull of unreviewed advisories
via REST provides ecosystem and patch timing data. The unreviewed set is used as a
held-out validation set, not for training.

---

## How to run

### Step 1 — Set up your environment

See `PREREQUISITES.md` for the full list of required packages and how to install them.

### Step 2 — API keys (optional but strongly recommended)

NVD has rate limits. Without a key, each page request waits 6.5 seconds. With a key,
it drops to under one second.

```bash
# Windows
$env:NVD_API_KEY = "your-nvd-key-here"
$env:GITHUB_TOKEN = "your-github-pat-here"

# Mac / Linux
export NVD_API_KEY="your-nvd-key-here"
export GITHUB_TOKEN="your-github-pat-here"
```

Get a free NVD key: https://nvd.nist.gov/developers/request-an-api-key  
Get a free GitHub PAT: https://github.com/settings/tokens (scope: read:public_repo)

### Step 3 — Run the notebook

Open `CVE_RiskClassification_v4_Final.ipynb` in Jupyter and run cells in sequence
from the top. The notebook is self-contained and creates all required directories
on first run.

If you want a quick end-to-end test before committing to a full NVD pull, set
`max_pages=2` in the `ingest_nvd()` call in Cell 7. That gives you around 1,000
records and runs in under a minute.

### Step 4 — Cache behaviour

The notebook caches the NVD and GitHub data as Parquet files in `data/raw/`.
On subsequent runs it loads from cache rather than hitting the APIs again.

To refresh the data, delete the relevant file:

```bash
# Refresh NVD data
del data\raw\nvd_cves.parquet          # Windows
rm data/raw/nvd_cves.parquet           # Mac/Linux

# Refresh GitHub data (deletes all three GitHub files)
del data\raw\github_advisories.parquet
del data\raw\github_unreviewed.parquet
del data\raw\github_all.parquet
```

---

## Pipeline overview

```
NVD REST API          GitHub GraphQL/REST
      │                       │
      ▼                       ▼
 nvd_cves.parquet    github_advisories.parquet
      │                       │
      └──────────┬────────────┘
                 ▼
         DuckDB — quality audit
         (missing values, duplicates,
          range checks, temporal anomalies)
                 │
                 ▼
         DuckDB — clean and filter
         317,522 records retained
                 │
                 ▼
         PySpark — LEFT JOIN on cve_id
         4.4% match rate
                 │
                 ▼
         PySpark — structural features
         (CWE count, ecosystem flags,
          patch timing, NVD scoring status)
                 │
                 ▼
         scikit-learn — TF-IDF
         30 unigrams and bigrams
         from description text
                 │
                 ▼
         features.parquet
         317,522 rows × 40 features
                 │
                 ▼
         Model training
         (Logistic Regression, Random Forest,
          LightGBM, XGBoost — temporal split)
```

---

## Key results

| Model | Test F1-Macro | Test Accuracy | Critical Recall |
|-------|--------------|---------------|-----------------|
| Logistic Regression | 0.226 | 0.256 | — |
| Random Forest | 0.281 | 0.316 | — |
| LightGBM | 0.300 | 0.340 | — |
| XGBoost (primary) | 0.297 | 0.336 | **0.77** |

The most important single number is Critical recall at 0.77. The model correctly
flags approximately three quarters of genuine Critical CVEs before NVD assigns an
official score. The F1-macro of 0.297 sits below the 0.70 target, but that gap
reflects the fundamental difficulty of predicting severity without the information
NVD analysts contribute — not a broken pipeline.

---

## Known limitations

**GitHub match rate is 4.4%.** Ecosystem and patch features are zero for 95.6% of
training records. Increasing the GitHub advisory coverage is the single highest-value
improvement available.

**TF-IDF depends on the NVD description column.** If the NVD Parquet cache was
built from an older version of the notebook that stored `description_length` instead
of the actual text, TF-IDF features will all be zero. Delete `data/raw/nvd_cves.parquet`
and re-run Cell 7 if you see `Total features: 10` instead of `Total features: 40`.

**No hyperparameter search was run.** XGBoost parameters are sensible defaults.
Running Optuna or GridSearchCV once the feature pipeline is stable would likely
add 3–5 points of F1-macro.

---

## Reproducibility

All results were produced on:
- Python 3.13.9 — Anaconda, Windows 11
- Random seed: 42
- Training: 2000-01-01 to 2022-12-31 (190,384 records)
- Validation: 2023-01-01 to 2023-12-31 (28,817 records)
- Test: 2024-01-01 to 2026-03-29 (98,321 records)

The full reproducibility log is written to `reports/reproducibility.json` at the
end of each notebook run.
