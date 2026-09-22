# db-delay-analysis — German Railway Delay Analysis & Dashboard

Two connected papers and one dashboard, built on a single shared data
pipeline over real Deutsche Bahn delay data.

1. **Paper 1 (main) — Station-level delay analysis.** Which stations and
   routes are most affected by delays, when, and why — an applied
   issues-and-solutions study: identify the worst-affected stations/routes,
   characterize the patterns (time of day, train type, seasonality,
   propagation from upstream stations), and propose concrete,
   evidence-backed recommendations.
2. **Paper 2 (secondary) — Evaluation leakage.** A shorter, methodological
   companion piece: naive random train/test splits silently inflate
   reported accuracy in delay-prediction models; quantified on the same
   dataset, with a short audit of how existing literature reports its own
   split methodology.
3. **Dashboard.** A Streamlit app: a map of Germany showing delay
   hotspots, station/route rankings, time trends — visualizing Paper 1's
   findings directly — plus a panel showing Paper 2's leakage result.

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

**Known caveats** (from the dataset's own README — matter for the analysis,
both papers, and the dashboard):
- Full station coverage only from **2025-11-02** onward; before that, only
  the ~100 biggest stations are included. "Most affected stations" claims
  must state which coverage regime they're computed over.
- Documented collection gaps (~98.9% file-level coverage overall) —
  missing hours are NOT the same as "no train ran" — don't let gaps read
  as artificially low delay for a station/period.
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
├── src/                          # shared pipeline, used by both papers + dashboard
│   ├── data_loader.py            # DuckDB queries against raw parquet
│   ├── cleaning.py
│   ├── features.py                # time-safe feature engineering
│   ├── split.py                   # naive random split vs time-based split (paper 2)
│   ├── model.py
│   └── evaluate.py                # MAE/RMSE, bootstrap CI, significance test
├── analysis/
│   ├── 01_station_rankings.py     # worst-affected stations/routes
│   ├── 02_temporal_patterns.py    # time-of-day, weekday, seasonality
│   ├── 03_propagation_analysis.py # upstream-delay effects
│   └── results/                   # saved tables/figures — evidence for paper 1
├── experiments/
│   ├── 01_leakage_comparison.py   # naive vs. correct split (paper 2's core result)
│   ├── 02_seed_robustness.py
│   ├── 03_breakdown_by_segment.py
│   └── results/                   # saved metrics — evidence for paper 2
├── notebooks/                     # exploration only; logic graduates into src/
├── papers/
│   ├── paper1_station_analysis/
│   │   ├── figures/
│   │   ├── references.bib
│   │   └── paper.md
│   └── paper2_leakage/
│       ├── literature_audit.csv   # paper | dataset | split method | disclosed?
│       ├── figures/
│       ├── references.bib
│       └── paper.md
├── dashboard/
│   ├── app.py
│   ├── components/                # map_view, rankings, trends, leakage_panel
│   └── data_prep.py
├── tests/
├── .gitignore
├── pyproject.toml / uv.lock
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
      ids, spot-check a single train's route)
- [ ] Cleaning step
- [ ] Station/route delay rankings (Paper 1 core result)
- [ ] Temporal pattern analysis (time-of-day, weekday, season)
- [ ] Propagation analysis (upstream-station effects)
- [ ] Time-safe feature engineering
- [ ] Naive vs. correct split comparison (Paper 2 core result)
- [ ] Seed robustness + statistical significance (Paper 2)
- [ ] Literature split-protocol audit table (Paper 2)
- [ ] Dashboard (map, rankings, trends, leakage panel)
- [ ] Paper 1 writeup
- [ ] Paper 2 writeup

## Roadmap

**Phase 1 — Ground Paper 1.** Explore the real data, clean it, compute
station/route delay rankings and temporal patterns. This is the main
paper's evidence and doesn't depend on anything else being built first.

**Phase 2 — Propagation + solutions.** Analyze upstream-delay effects,
turn the findings into concrete, evidence-backed recommendations
(the "solutions" half of Paper 1).

**Phase 3 — Paper 2 (leakage).** Build the naive-vs-correct split
comparison on the same cleaned data, add bootstrap CI/significance
testing, and the literature audit table.

**Phase 4 — Dashboard.** Map, station/route rankings, time trends
(Paper 1), and a panel visualizing the leakage result (Paper 2).

**Phase 5 — Write-up.** Both papers, using only numbers that trace back
to saved files in `analysis/results/` and `experiments/results/`.

## Notes to self

- Test cleaning/feature logic on a small DuckDB-filtered sample before
  running against a full month.
- Never commit raw or processed data to git.
- Every number that ends up in either paper should trace back to a saved
  file in `analysis/results/` or `experiments/results/`, not just
  terminal output.
- State the coverage regime (pre/post 2025-11-02) explicitly whenever
  comparing stations — otherwise "most affected" rankings are comparing
  apples to oranges.