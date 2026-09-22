# DB Delay Prediction — Evaluation Leakage in Train Delay Forecasting

A German-railway case study showing that naive (random) train/test splits
silently inflate reported accuracy in delay-prediction models, with a
supporting dashboard visualizing the finding.

Two connected deliverables, one shared data pipeline:

1. **Paper** — quantifies the accuracy gap between a naive random split and
   a correct time-based split on real Deutsche Bahn data, and audits how
   existing train-delay-prediction literature reports its own split
   methodology.
2. **Dashboard** — a Streamlit app visualizing delays across German
   stations (map, rankings, trends) plus a panel showing the leakage
   result itself.

All pipeline code is written from scratch as part of this project —
no external ML pipelines or boilerplate templates.

## Data

Source: [piebro/deutsche-bahn-data](https://huggingface.co/datasets/piebro/deutsche-bahn-data)
(public, CC BY 4.0, collected from Deutsche Bahn's Timetables API).

Monthly parquet files under `data/raw/`, downloaded via:

```bash
uv run --with "huggingface-hub" hf download piebro/deutsche-bahn-data \
    --repo-type=dataset --local-dir=./data/raw --include "monthly_processed_data/*"
```

**Known caveats** (from the dataset's own README — matter for both the
cleaning step and the paper's data section):
- Full station coverage only from **2025-11-02** onward; before that, only
  the ~100 biggest stations are included. Don't mix regimes without
  accounting for it.
- Documented collection gaps (~98.9% file-level coverage overall) —
  missing hours are NOT the same as "no train ran."
- Timestamps are naive local time (Europe/Berlin) — watch DST transitions.
- `line_number` is null for long-distance trains (ICE/IC/EC); only
  populated for regional services.

Station coordinates (for the dashboard map) come from
[trainline-eu/stations](https://github.com/trainline-eu/stations),
joined on `db_id` == the dataset's `eva` column.

## Folder structure

```
db-delay-project/
├── data/
│   ├── raw/                      # downloaded monthly parquet (gitignored)
│   ├── processed/                # cleaned/filtered slices (gitignored)
│   └── external/
│       └── stations_de.csv       # station -> lat/lon lookup
├── src/                          # shared pipeline, used by experiments + dashboard
│   ├── data_loader.py            # DuckDB queries against raw parquet
│   ├── cleaning.py
│   ├── features.py                # time-safe feature engineering
│   ├── split.py                   # naive random split vs time-based split
│   ├── model.py
│   └── evaluate.py                # MAE/RMSE, bootstrap CI, significance test
├── experiments/
│   ├── 01_leakage_comparison.py
│   ├── 02_seed_robustness.py
│   ├── 03_breakdown_by_segment.py
│   └── results/                   # saved metrics — the evidence behind the paper
├── notebooks/                     # exploration only; logic graduates into src/
├── paper/
│   ├── literature_audit.csv       # paper | dataset | split method | disclosed?
│   ├── figures/
│   ├── references.bib
│   └── paper.md
├── dashboard/
│   ├── app.py
│   ├── components/                # map_view, rankings, trends, leakage_panel
│   └── data_prep.py
├── tests/
├── .gitignore
├── requirements.txt
└── README.md
```

## Setup

Dependency management via [uv](https://docs.astral.sh/uv/) — `pyproject.toml`
and `uv.lock` are the source of truth for dependencies (both committed to
git, unlike the data folders).

```bash
cd db-delay-project
uv init --no-readme          # only if pyproject.toml doesn't exist yet
uv add duckdb pandas lightgbm scikit-learn pyarrow shap streamlit plotly
```

Run anything with `uv run`, e.g.:

```bash
uv run python -c "import duckdb; print(duckdb.__version__)"
```

No manual venv activation needed — `uv run` handles it per-command.

DuckDB is used to query the raw parquet files directly (no full in-memory
load needed — important on limited hardware). Pandas/LightGBM take over
once data is filtered down to a working size.

## Status

- [x] Dataset downloaded (monthly parquet)
- [ ] Data exploration pass (row counts, date range, station coverage,
      delay distribution, cancellation rate, missing values, duplicate
      ids, spot-check a single train's route) — in progress
- [ ] Cleaning step
- [ ] Time-safe feature engineering
- [ ] Naive vs. correct split comparison (core experiment)
- [ ] Seed robustness + statistical significance
- [ ] Breakdown by train type / route density
- [ ] Literature split-protocol audit table
- [ ] Dashboard (map, rankings, trends, leakage panel)
- [ ] Paper writeup

## Roadmap

**Phase 1 — Ground the claim.** Explore the real data, build cleaning and
time-safe features, run the naive-vs-correct split comparison. Nothing
else is worth building until this shows a real, consistent effect.

**Phase 2 — Make it rigorous.** Bootstrap confidence intervals,
significance testing, segment breakdowns, literature audit table.

**Phase 3 — Dashboard.** Map, station/route rankings, time trends, and a
panel visualizing the Phase 1–2 result directly.

**Phase 4 — Write-up.** Intro, related work, data, method, results
(audit table + experiment), discussion, limitations.