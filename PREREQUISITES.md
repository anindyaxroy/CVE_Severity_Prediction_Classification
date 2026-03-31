# Prerequisites

Everything you need to run the CVE severity classifier notebook from scratch.
Read this before running anything.

---

## Python version

Python 3.10 or above is required. The notebook was built and tested on **Python 3.13.9**.
Earlier versions may work but have not been tested.

Check what you have:

```bash
python --version
```

If you need to install or upgrade Python, use Anaconda (recommended) or the official
installer at https://www.python.org/downloads/

---

## Java (required for PySpark)

PySpark needs Java 8 or above on your PATH. The notebook uses PySpark for the
NVD–GitHub join step. Without Java, PySpark will fail to start.

Check:
```bash
java -version
```

If Java is missing, install the latest OpenJDK from https://adoptium.net/
After installing, restart your terminal and check again before proceeding.

On Windows you may also need to set `JAVA_HOME`:
```
System Properties → Environment Variables → New System Variable
Name:  JAVA_HOME
Value: C:\Program Files\Eclipse Adoptium\jdk-21.0.x.x-hotspot  (adjust to your path)
```

---

## Package installation

Install everything in one go:

```bash
pip install duckdb pyspark pandas numpy pyarrow requests tqdm
pip install scikit-learn xgboost lightgbm shap
pip install matplotlib seaborn joblib
pip install imbalanced-learn networkx
pip install jupyter jupyterlab ipywidgets
```

Or install from the list below one group at a time if you prefer to understand
what each package does.

---

## Package reference

### Data and pipeline

| Package | Version tested | Purpose |
|---------|---------------|---------|
| duckdb | 0.10.x | In-process SQL — quality audit and cleaning |
| pyspark | 3.5.1 | Distributed join and feature engineering |
| pandas | 2.x | Feature matrix assembly and analysis |
| numpy | 1.x / 2.x | Numerical operations throughout |
| pyarrow | 14.x+ | Parquet file read and write |
| requests | 2.x | NVD REST API and GitHub REST API calls |
| tqdm | 4.x | Progress bars during API ingestion |

### Machine learning

| Package | Version tested | Purpose |
|---------|---------------|---------|
| scikit-learn | 1.x | TF-IDF vectoriser, Logistic Regression, Random Forest, metrics |
| xgboost | 2.x | Primary classifier with early stopping and SHAP support |
| lightgbm | 4.x | Comparative gradient boosting model |
| shap | 0.4x | Feature importance and explainability for XGBoost |
| imbalanced-learn | 0.12.x | Available but not used — SMOTE was evaluated and rejected |

### Visualisation

| Package | Version tested | Purpose |
|---------|---------------|---------|
| matplotlib | 3.x | All chart generation |
| seaborn | 0.13.x | Correlation heatmap |
| joblib | 1.x | Model serialisation and loading |

### Environment

| Package | Version tested | Purpose |
|---------|---------------|---------|
| jupyter | 7.x | Notebook interface |
| jupyterlab | 4.x | Optional — preferred interface |
| ipywidgets | 8.x | Progress display in notebooks |
| networkx | 3.x | Imported in environment check — not used in current pipeline |

---

## Verification

Once installed, run Cell 3 in the notebook. It imports every package and tells you
immediately if anything is missing:

```
  OK   duckdb
  OK   pyspark
  OK   pandas
  OK   numpy
  OK   requests
  OK   xgboost
  OK   sklearn
  OK   shap
  OK   matplotlib
  OK   seaborn
  OK   networkx
  OK   imblearn
  OK   tqdm
  OK   joblib
  OK   lightgbm

All packages ready. Proceed to Section 1.
```

If any line shows MISSING, install that package and re-run the cell before continuing.

---

## API access

### NVD API key (optional but strongly recommended)

Without a key, NVD rate-limits you to 5 requests per 30 seconds — the notebook
adds a 6.5-second sleep between pages. A full pull takes around 2.5 hours.

With a key, the limit rises to 50 requests per 30 seconds and the sleep drops
to 0.7 seconds. A full pull takes around 15 minutes.

Register free at: https://nvd.nist.gov/developers/request-an-api-key  
Keys activate within one hour of registration.

```bash
# Set before launching Jupyter
$env:NVD_API_KEY = "your-key-here"        # Windows PowerShell
export NVD_API_KEY="your-key-here"         # Mac / Linux
```

### GitHub Personal Access Token (required for full data pull)

The GitHub GraphQL API requires authentication to pull the full advisory dataset.
Without a token, the GraphQL call will fail and the notebook falls back to the
REST endpoint, which has a much lower rate limit.

Create a free token at: https://github.com/settings/tokens  
Required scope: **read:public_repo** (public advisories require no private scope)

```bash
$env:GITHUB_TOKEN = "github_pat_..."       # Windows PowerShell
export GITHUB_TOKEN="github_pat_..."       # Mac / Linux
```

Never paste your token directly into the notebook. Always load it via
`os.getenv("GITHUB_TOKEN")`. The notebook is set up this way already — do not
change it.

---

## Memory requirements

The notebook was tested on a machine with 32 GB RAM. The minimum practical
requirement is around 16 GB. The PySpark session is configured to use 4 GB of
driver memory. If your machine has less than 16 GB you may need to reduce this:

In Cell 25, change:
```python
.config("spark.driver.memory", "4g")
```
to:
```python
.config("spark.driver.memory", "2g")
```

The feature matrix at 317,522 rows by 40 columns fits comfortably in memory
on most modern laptops. The TF-IDF sparse matrix is the largest object in the
pipeline and peaks at around 2 GB before being converted to a dense Parquet file.

---

## Tested operating systems

| OS | Status |
|----|--------|
| Windows 11 | Tested — primary development environment |
| macOS 14 (Sonoma) | Should work — no known incompatibilities |
| Ubuntu 22.04 | Should work — PySpark is well supported on Linux |

On macOS you may need to set `OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES` before
starting Jupyter if PySpark throws a fork safety warning:

```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
jupyter lab
```

---

## Troubleshooting

**PySpark fails to start**  
Almost always a Java issue. Run `java -version` in a fresh terminal. If it prints
nothing, install Java and restart.

**TF-IDF shows 0 features**  
The NVD cache was built without the description column. Delete
`data/raw/nvd_cves.parquet` and re-run Cell 7.

**GitHub GraphQL returns empty**  
Check that `GITHUB_TOKEN` is set in the same terminal session where you launched
Jupyter. Token must be set before Jupyter starts — setting it in a separate
terminal window does not help.

**DuckDB error on re-run**  
The DuckDB file path includes a timestamp, so each run creates a new database.
Old database files in `data/staging/` are harmless and can be deleted to save disk space.

**Memory error during PySpark join**  
Reduce `spark.driver.memory` in Cell 25 and restart the kernel before re-running.
