# db-delay-analysis — German Railway Delay Analysis & Dashboard

Two connected papers and one dashboard, built on a single shared data
pipeline over real Deutsche Bahn delay data.

1. **Paper 1 (main) — Station-level delay analysis + prediction model.**
   Which stations and routes are most affected by delays, when, and why —
   plus a predictive model that estimates major-delay risk. Three parts:
   - **Delay category prediction (the main ML model)** — classify a given
     train/station situation as On time / Minor delay / Major delay, and
     output a probability of major delay rather than just a label.
   - **Station Reliability Ranking** — a composite score per station from
     average delay + cancellation rate, to answer: which stations create
     delays, which recover quickly, which are bottlenecks.
   - **Train Line Performance Analysis** — per `train_type`/`train_number`/
     `line_number`: trip count, average delay, cancellation rate, worst
     station on that line.
2. **Paper 2 (secondary) — Evaluation leakage.** A shorter, methodological
   companion piece: naive random train/test splits silently inflate
   reported accuracy in delay-prediction models; quantified on the same
   dataset (reusing the Paper 1 model as the test case), with a short
   audit of how existing literature reports its own split methodology.
3. **Dashboard.** A Streamlit app: a map of Germany showing delay
   hotspots, the station reliability ranking, train-line performance
   pages, an interactive "predict my train's delay risk" panel (the
   model from Paper 1), and a panel showing Paper 2's leakage result.

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
  the ~100 biggest stations are included. "Most affected stations" and
  reliability-score claims must state which coverage regime they're
  computed over.
- Documented collection gaps (~98.9% file-level coverage overall) —
  missing hours are NOT the same as "no train ran" — don't let gaps read
  as artificially low delay/cancellation for a station or line.
- Timestamps are naive local time (Europe/Berlin) — watch DST transitions.
- `line_number` is null for long-distance trains (ICE/IC/EC); only
  populated for regional services — the train-line analysis needs to
  handle these two cases separately (group ICE/IC by `train_number`
  instead).

Station coordinates (for the dashboard map) come from
[trainline-eu/stations](https://github.com/trainline-eu/stations),
joined on `db_id` == the dataset's `eva` column.

## `delay_category` definition (used throughout)

| `delay_in_min` | Category |
|---|---|
| 0–5 | On time |
| 6–15 | Minor delay |
| >15 | Major delay |

Cancellations (`is_canceled`) are tracked as a separate flag, not folded
into this scale — a cancellation isn't "infinite delay," it's a different
failure mode with its own rate.

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
│   ├── model.py                   # delay_category classifier
│   ├── reliability_score.py       # station reliability scoring formula
│   └── evaluate.py                # metrics, bootstrap CI, significance test
├── analysis/
│   ├── 01_station_rankings.py     # station reliability score + rankings
│   ├── 02_temporal_patterns.py    # time-of-day, weekday, seasonality
│   ├── 03_propagation_analysis.py # upstream-delay effects
│   ├── 04_train_line_performance.py  # per train_type/number/line stats
│   └── results/                   # saved tables/figures — evidence for paper 1
├── models/
│   └── delay_category_model/      # trained classifier + evaluation report
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
│   ├── components/
│   │   ├── map_view.py            # delay hotspot map
│   │   ├── station_reliability.py # reliability ranking table
│   │   ├── train_line_performance.py
│   │   ├── delay_risk_predictor.py # interactive delay_category prediction
│   │   ├── trends.py
│   │   └── leakage_panel.py
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
- [ ] `delay_category` labeling + class balance check
- [ ] Station Reliability Ranking (score formula, per-station table)
- [ ] Train Line Performance Analysis (per type/number/line table)
- [ ] Temporal pattern analysis (time-of-day, weekday, season)
- [ ] Propagation analysis (upstream-station effects)
- [ ] Time-safe feature engineering
- [ ] Delay category classifier (main ML model) — trained + evaluated
- [ ] Naive vs. correct split comparison (Paper 2 core result, reusing the
      classifier above as the test case)
- [ ] Seed robustness + statistical significance (Paper 2)
- [ ] Literature split-protocol audit table (Paper 2)
- [ ] Dashboard (map, reliability ranking, train-line pages, delay-risk
      predictor, trends, leakage panel)
- [ ] Paper 1 writeup
- [ ] Paper 2 writeup

## Roadmap

**Phase 1 — Ground Paper 1's descriptive side.** Explore and clean the
real data, define `delay_category`, compute the Station Reliability Score
and Train Line Performance tables. This is evidence that doesn't depend
on any model existing yet.

**Phase 2 — The prediction model.** Time-safe features, train the
delay-category classifier, evaluate properly (per-class precision/recall,
not just accuracy — classes are imbalanced since major delays are rarer).
This is the main ML deliverable.

**Phase 3 — Propagation + solutions.** Analyze upstream-delay effects,
turn the rankings + model findings into concrete, evidence-backed
recommendations (the "solutions" half of Paper 1).

**Phase 4 — Paper 2 (leakage).** Reuse the Phase 2 classifier: compare
naive-random vs. correct time-based split on it, add bootstrap CI/
significance testing, and the literature audit table.

**Phase 5 — Dashboard.** Map, reliability ranking, train-line performance
pages, interactive delay-risk predictor (Phase 2's model), trends, and the
leakage panel (Phase 4's result).

**Phase 6 — Write-up.** Both papers, using only numbers that trace back
to saved files in `analysis/results/`, `models/`, or `experiments/results/`.

## Notes to self

- Test cleaning/feature logic on a small DuckDB-filtered sample before
  running against a full month.
- Never commit raw or processed data to git.
- Every number that ends up in either paper should trace back to a saved
  file, not just terminal output.
- State the coverage regime (pre/post 2025-11-02) explicitly whenever
  comparing stations or lines — otherwise rankings compare apples to
  oranges.
- `delay_category` classes are almost certainly imbalanced (most trains
  are on time or slightly late; major delays are rare) — accuracy alone
  will be misleading; use precision/recall/F1 per class, and consider
  class weighting.
- Decide the Station Reliability Score formula deliberately and document
  it (e.g. weighted combination of avg delay + cancellation rate,
  normalized) — write down the exact formula before computing it so it's
  reproducible and defensible in the paper.