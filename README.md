# Automated CSV Profiler and Verified Insight Generator

CSCI 5502 AI-Enhanced Data Mining, Assignment #2 ("From CSV to Evidence")

```
CSV file → Python analysis → verified summary (facts) → LLM explanation → verification → generated report
```

Python computes every statistic. The language model only rewords numbered, Python-written facts. A verifier then rejects any LLM sentence that contains a number not in those facts, names an unknown column, or uses causal language.

---

## 1. Purpose

The program accepts any reasonably well-formed, single-table CSV and automatically produces a profile of it:

1. **Dataset overview.** Filename, rows and columns, and for every column its inferred technical type, non-missing count, missing %, unique count and probable role. The role is one of: numeric measure, categorical, boolean, date-like, identifier-like, free-text, or unknown/mixed.
2. **Data-quality profile:**
   - duplicate rows
   - constant and empty columns
   - high missingness (threshold **30%**)
   - mixed or inconsistent types and formats, including thousands separators and categories that differ only by letter case
   - identifier-like and high-cardinality columns
   - potentially sensitive fields (by name and by value pattern)
   - suspected missing-value codes (e.g. `-999`)
   - zero-inflated columns
   - unit-of-measure columns
   - dominant categories
   - disguised missing tokens
3. **Descriptive statistics by role:**
   - Numeric: count, missing, min, max, mean, median, mode (when meaningful), std, Q1, Q3, IQR, and 1.5×IQR outliers.
   - Categorical: top-10 counts and percentages.
4. **Relationships:**
   - Pearson and Spearman correlation matrix over numeric *measures* only.
   - Strongest positive and negative pairs.
   - Numeric-by-category association (eta²).
   - Trends over time.
5. **Adaptive visualizations** (at most 10), chosen from the column roles. Every plot type that is not produced is listed with the reason.
6. **5–8 evidence-based insights**, written by a local Ollama model from verified facts, then machine-checked.

Nothing is ever deleted, imputed or "cleaned". The program only flags problems.

## 2. Python version

Developed and tested with **Python 3.12**. The code was also checked against pandas 2.2 and pandas 3.0 and gave identical results. Python 3.10 or newer is required.

## 3. Required libraries

| Library | Used for |
|---|---|
| pandas ≥ 2.2 | loading, profiling, statistics |
| numpy ≥ 1.26 | numeric helpers |
| matplotlib ≥ 3.8 | all plots |
| Ollama (separate app, optional) | local LLM for the narrative insights |

The LLM is called through Python's built-in `urllib`, so no extra package and no API key is needed. No automated-profiling package (ydata-profiling, Sweetviz, AutoViz) is used. Every check is implemented in `src/`.

## 4. Installation

```bash
git clone <this repository>
cd automated_csv_profiler
python -m venv .venv
# Windows:  .venv\Scripts\activate      macOS/Linux:  source .venv/bin/activate
pip install -r requirements.txt
```

Optional, for the AI insights:
1. Install Ollama from https://ollama.com.
2. Pull the model: `ollama pull llama3.1:8b`
3. Make sure the Ollama app, or `ollama serve`, is running.

## 5. Running the program

Put the CSV files in `data/` (see `data/README.md`), then run these from the repository root.

```bash
# Dataset A: Washington EV population
python src/profiler.py --csv data/ev_population_data.csv --name dataset_a

# Dataset B: NYSERDA Utility Energy Registry: same code, only the path and folder name change
python src/profiler.py --csv data/nyserda_uer_county_monthly.csv --name dataset_b
```

From Python or a notebook:

```python
import sys; sys.path.insert(0, "src")
from profiler import generate_profile
generate_profile("data/ev_population_data.csv", output_dir="output", use_llm=True, dataset_name="dataset_a")
```

Each run writes `output/<name>/` containing:
- `report.md`
- `column_profile.csv`
- `analysis_summary.json`
- `plots/plot_01.png` and onward
- `llm_prompt.txt`
- `llm_response.txt`

A run on either dataset (~300,000 rows) takes about 12 seconds, plus the LLM time.

## 6. How to select a CSV

Change only `--csv` (and optionally `--name`). There are no dataset-specific branches and no hard-coded column names. All behaviour comes from inferred column roles and the documented thresholds in `src/config.py`.

| Option | Default | Meaning |
|---|---|---|
| `--csv` | (required) | path to the CSV |
| `--out` | `output` | parent output folder |
| `--name` | CSV file name | output sub-folder name |
| `--no-llm` | off | skip the LLM stage |
| `--model` | `llama3.1:8b` | Ollama model name |
| `--missing-threshold` | `30` | high-missing threshold, % |
| `--max-plots` | `10` | maximum number of plots |
| `--top-n` | `10` | categories listed per column |

Environment variable `OLLAMA_HOST` overrides the Ollama address (default `http://localhost:11434`).

## 7. Enabling or disabling the LLM

- **Enabled (default):** the program checks that Ollama is running and that the model is installed. It then sends the prompt in `llm_prompt.txt` and saves the raw reply to `llm_response.txt`.
- **Disabled:** add `--no-llm`, or pass `use_llm=False`.
- **Graceful failure:** if Ollama is off, the model is missing or the call fails, the program still writes the overview, quality results, statistics, plots and JSON summary. The report then states that *AI-generated narrative insights were skipped because the model was unavailable* and gives the reason. The insight section is instead filled with deterministic, clearly labelled template sentences built directly from the Python facts.

### How the LLM is kept honest

1. Python writes `analysis_summary.json` and turns it into numbered **facts** (`F01`, `F02`, …). Each fact is a sentence that Python composed from values it computed.
2. The prompt contains only these facts. The raw CSV is never sent. The model must return JSON insights that cite fact IDs. Temperature is 0.
3. `src/verify.py` checks every insight:
   - Every number must match a number in the cited facts, allowing only for normal rounding.
   - Every `backticked` name must be a real column.
   - Causal wording ("causes", "leads to", "due to", …) is rejected.
4. Accepted insights appear in the report as "LLM, verified". Rejected ones are listed with their reasons in the report's verification log. If a required insight category is missing, a labelled template insight fills the gap.

## 8. Model used

**Ollama `llama3.1:8b`** (temperature 0, seed 42, JSON output mode).
*(If you run a different model, update this line; the model name is also recorded in every `llm_prompt.txt` and report.)*

## 9. Known limitations

- Type and role inference are heuristics. The thresholds live in `src/config.py`, and unusual columns can be misclassified. For example, a sequential integer measure could look like an ID.
- Column meanings, units and valid ranges are never inferred. The system flags a *unit* column when one exists, but it cannot tell which numeric column the units apply to.
- Suspected missing-value codes (e.g. `-999`) are flagged but left in the headline statistics, because the data are never altered. A supplementary "excluding codes" view is shown beside them.
- Outlier flags come from the 1.5×IQR rule. On zero-heavy data (IQR ≈ 0) this rule flags many legitimate values.
- Several checks use random samples with a fixed seed (type inference 2,000 values, sensitive patterns, eta² on 50,000 rows, scatterplots on 5,000 points), so results are reproducible but approximate.
- The sensitive-data check uses only name and pattern heuristics. The absence of a warning does not mean a dataset is safe.
- The LLM verifier checks numbers, column names and causal words. It cannot judge whether an interpretation is *sensible*, so the insights still need a human read.
- Supported inputs are single-table CSVs that fit in memory. Excel, JSON, geospatial formats and streaming are out of scope. Coordinate text is recognised but not analysed.

## 10. Data sources

| | Dataset A (development) | Dataset B (new) |
|---|---|---|
| Agency / organization | Washington State Department of Licensing (published on Data.WA) | New York State Energy Research and Development Authority (NYSERDA), on data.ny.gov |
| Title | Electric Vehicle Population Data | Utility Energy Registry Monthly County Energy Use: Beginning 2021 |
| Source link | https://data.wa.gov/Transportation/Electric-Vehicle-Population-Data/f6w7-q2d2 (also https://catalog.data.gov/dataset/electric-vehicle-population-data) | https://data.ny.gov/Energy-Environment/Utility-Energy-Registry-Monthly-County-Energy-Use-/46pe-aat9 |
| Date accessed | 2026-09-28 (course copy `20260126_Electric_Vehicle_Population_Data.csv` from the CSCI 5502 Canvas module) | 2026-09-28 |
| Shape | 299,705 rows × 16 columns | 299,662 rows × 14 columns |
| Structure | mostly categorical; codes stored as numbers; one numeric measure; model-year dates | numeric values with thousands separators; mixed units; `-999` privacy codes; year + month time series |

Both datasets are public and aggregated. No sensitive, confidential or restricted data are used.

## Repository layout

```
automated_csv_profiler/
├── README.md            this file
├── requirements.txt
├── src/
│   ├── profiler.py      entry point: generate_profile() + command line
│   ├── config.py        every threshold, with its justification
│   ├── loader.py        reads the CSV as text (encoding / delimiter fallback)
│   ├── inference.py     technical type + probable role per column
│   ├── quality.py       data-quality checks (flag, never modify)
│   ├── stats.py         descriptive statistics by role, 1.5×IQR outliers
│   ├── relationships.py correlation, eta², time trend
│   ├── viz.py           adaptive plot planning and rendering
│   ├── facts.py         numbered verified facts + template insights
│   ├── llm.py           prompt building and the Ollama call
│   ├── verify.py        checks LLM output against the facts
│   ├── report.py        builds report.md from the summary
│   ├── utils.py
│   └── tests/
│       ├── run_edge_cases.py   8 awkward CSVs + LLM-down test
│       └── mock_ollama.py      fake Ollama server for testing the verifier
├── data/README.md       how to obtain the two CSVs
├── output/dataset_a/    generated for Dataset A
├── output/dataset_b/    generated for Dataset B
├── video/video_link.txt
└── logs/genai_log.md
```

**Tests:** `python src/tests/run_edge_cases.py` profiles 7 generated edge-case CSVs (all text, numeric only, a single column, a messy file, a tiny file, cp1252 encoding, and `;`-delimited) plus an LLM-unavailable run. It checks that every required output is produced.

**Security:** the repository contains no API keys, tokens or credentials. Ollama runs locally and needs none.
