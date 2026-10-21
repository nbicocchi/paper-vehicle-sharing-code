# Bike-sharing and cycling counts: reproducible Bologna study

Code and numerical outputs supporting `../paper-vehicle-sharing-text/main.pdf`.
Original datasets are preserved under `dataset/` and are **not redistributed**.

## Reproduce

Tested with Python 3.14 on Linux; exact versions and file hashes are recorded in
`results/reproducibility.json`. From this directory:

```bash
python -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python code/run.py
```

The full runner executes preparation, figures, diagnostics, counter prediction,
synthetic experiments, LaTeX table export, and tests. It stops on failure.
The first run reads approximately 700 MB of sharing CSVs. Subsequent runs reuse
the de-identified trip cache only when input hashes and cohort version match.
For an explicit rebuild: `.venv/bin/python code/prepare.py --force`.

Compile the paper separately from the text directory:

```bash
latexmk -pdf -interaction=nonstopmode -halt-on-error main.tex
```

To run only the known-demand experiment (no empirical files required):

```bash
.venv/bin/python code/synthetic.py
```

## Inputs actually used

- `dataset/bike-sharing-2023.csv`, `dataset/bike-sharing-2024.csv`.
- `dataset/colonnine-conta-bici.csv`, restricted to local years 2023–2024.
- `dataset/meteo-bologna-ilmeteo-2019-2025.csv` (daily temperature and rain).

Car sharing, traffic loops, and air-quality files are outside this study.

## Pipeline

| Script | Purpose |
| --- | --- |
| `code/prepare.py` | Deduplicate, harmonise bike types, define task-specific validity, audit and group counters; handle DST exposure. |
| `code/analyse.py` | Temporal/type distributions, OD aggregates, activity/balance maps, and aggregated predictor features. `draw_trip_profiles` redraws Figures 1–3 with the maps' font settings and design width. |
| `code/citymap.py` | Matched 500 m activity and arrival–departure balance maps, with 06–10, 10–15, 15–20, and night 20–06 panels pooled over 2023–2024, over cached OpenStreetMap context. |
| `code/diagnostics.py` | Matched night fractions, OD zoning sensitivity, cleaning-population sensitivity, daily balance persistence. |
| `code/predict.py` | Two model families (boosting and regularised Poisson), each with the same four feature sets, plus separate uniform scaling; exclude each of four locations and test it in 2023 and 2024. |
| `code/error_analysis.py` | Per-location flow, sharing context, WAPE gains and signed bias; weekly observed/predicted 2024 profiles. Called by `predict.py`. |
| `code/synthetic.py` | Known-demand inverse problem, selective priors, direction/route/regularisation sensitivity, observational-equivalence witness. |
| `code/export_tables.py` | Generate manuscript tables/macros and provenance manifest. |
| `code/test_pipeline.py` | Deterministic and artifact-integrity tests. |

`code/dati_esterni/` is ignored by git and stores **sensitive local intermediates**.
Removing user/bicycle identifiers does not anonymise exact endpoints.
`results/` contains computed diagnostics and aggregates, not invented values.
Figures/tables are written to the sibling text repository. The unsuppressed OD
table remains in the local cache; the result CSV suppresses cells with fewer than
ten annual period-specific observations. This threshold is not a privacy guarantee.
No output should be published before permissions and disclosure risks are reviewed.

## Important findings and boundaries

- Every sharing analysis excludes recorded durations **below 30 seconds**; exactly
  30 seconds is retained. Missing, zero and negative durations are ineligible.
  Cache versioning enforces this rule throughout maps, counts and predictors.
- Almost all October 2023 source records have identical start/end timestamps and
  are therefore removed by the 30-second rule. The resulting near-empty month is
  flagged as lost coverage, not interpreted as zero demand. Periods with at least
  100 endpoint candidates and fewer than 1% surviving the duration filter are
  excluded from sharing/count comparisons. Raw audit counts remain available.
- The former counter-availability heatmap (old Figure 1) is no longer generated
  or included in the manuscript; the coverage table and numerical audit remain.
- The retained counter groups are **Ercolani, Manzi, Parri, Sabotino, San Donato,
  Sturzo, and Zanardi**. Laura Bassi, Massarenti, Mazzini, Murri, Orti, and Saragozza
  are explicitly excluded, including paired devices, throughout the counter pipeline.
  This user-requested selection is not a diagnosis of instrument failure.
- The prediction experiment uses **four** locations with 2023 coverage. Each is
  excluded from training and tested in 2023 and 2024. Other groups remain in the
  availability audit but are not used as an additional prediction test.
  Sharing endpoints are not filtered according to counter availability.
- Conflicting counter duplicates are discarded in pairs; absent/invalid observations
  are not zeros. Only complete service-day periods enter prediction.
- Night means 23:00–05:59, service day begins 06:00. Daylight-saving nights have
  their actual exposure. Rental departure times remain source-local/offset-naive.
- The predictors estimate counts with contemporaneous observed covariates. They do
  **not** forecast future trips before sharing/weather are observed.
- Both model families use all four cumulative feature sets: no sharing, citywide
  sharing, local endpoints, and OD geometry. There are eight fitted configurations
  plus a separate uniform-scaling baseline. The local-error figure shows all eight
  in matched rows (boosting/Poisson) and repeats scaling as a grey reference.
  Paired feature comparisons use each model family's own no-sharing baseline.
  Poisson regressions use fixed L2 penalty
  1, and training-only numeric imputation/standardisation and categorical encoding.
  They use exactly the same exposure weights and test rows as boosting. Their
  settings are not selected from test performance. Location diagnostics are in
  `results/prediction-site-diagnostics.csv`; weekly 2024 profiles are in
  `results/prediction-weekly-2024.csv`, covering the four original locations only.
  Weekly rates sum both directions and divide
  by available complete-period hours, without filling missing weeks with zeros.
- OD map connectors and chord-proximity features are **not routes**. There is no
  validated Bologna cycling network or empirical citywide OD matrix in this version.
- Figures 4 and 5 now show activity and balances, not OD connectors. Both pool
  2023–2024 data in four panels (06–10, 10–15, 15–20, night 20–06), with the same 500 m
  cells, map extent, valid windows, and 200-event display threshold. `activity.pdf` shows
  `(arrivals + departures) / valid hours` on a shared sequential scale;
  `balances.pdf` shows `(arrivals - departures) / (arrivals + departures)` on a
  shared diverging scale. Each endpoint uses its own event time. Compromised
  sharing periods are excluded from both maps: a window is retained only when
  every constituent hour belongs to a usable sharing service period and falls
  within the study years. Night belongs to its evening's service date. The maps'
  20–06 night does not change the 23–06 definition in OD comparisons or prediction.
  The hour denominator follows Europe/Rome clock changes. Selection, aggregates,
  and design are exported as `results/sharing-map-periods.csv`,
  `sharing-map-grid.csv`, and `sharing-map-design.json`. The old OD and six-hour
  rendering helpers remain available but are no longer called by the pipeline.
  Counter names are labelled only in the activity map's upper-left panel.
- Historical Overpass acquisition attempts returned HTTP 406/429. Even a successful
  download would still require access-rule, section/link, and route validation.
- The map uses present-day OpenStreetMap raster tiles only as cartographic context,
  not for routing. The first run downloads the small map
  viewport from `tile.openstreetmap.org`; subsequent runs use cached tiles in
  `code/dati_esterni/basemap/`. Attribution and tile provenance are recorded.
- The synthetic lattice is not Bologna. Its noise, sample selection, latent gravity
  attributes, route mixture, and fixed section placement are deliberately explicit.

## Before a journal submission

Confirm authorship and data-use rights; verify source documentation and the timestamp
defect with the provider; complete full-text literature checks; validate instrumentation;
obtain and check a historical bicycle network if empirical OD inference is pursued.
No claim of public data availability, ethical approval, a published DOI, or a fully
validated citywide demand reconstruction is made.
