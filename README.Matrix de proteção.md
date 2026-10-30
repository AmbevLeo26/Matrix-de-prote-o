# Protection Matrix (pandas version)

Rebuild of notebook `07.protection_matrix_analysis` in plain pandas: baseline analysis,
2026/27 lever simulation, group migration and volume capture. No Spark after the export.

```
protection_matrix/
├── config.yaml          ← every parameter; the only file you edit per scenario
├── pm.py                ← all logic (~350 lines, replaces ~6,000 lines of notebook 07)
├── plots.py             ← one heatmap + one bar function
├── run.py               ← runs everything, writes output/
├── make_demo.py         ← fake data with the real columns (python run.py --demo)
├── sql/
│   ├── 00_export_inputs.py    ← run in Databricks: builds base_analise from raw UC tables
│   └── 01_volume_protect.sql  ← replaces notebooks 02-04 for the next refresh
├── inputs/sopi_quotas.csv     ← SOPI December quotas copied from notebook 07
└── tests/
    ├── test_pm.py             ← 8 tests incl. the 20 hl worked example
    └── check_export_spark.py  ← runs the export SQL on mock tables (needs pyspark)
```

## The day

| When | Step | Done when |
|---|---|---|
| 09:00 | `pip install -r requirements.txt`, then `python run.py --demo` and `pytest -q tests` | demo charts in `output_demo/`, 8 tests pass |
| 09:30 | **Decide the 5 `DECIDE` lines in `config.yaml`** (see below) with whoever owns the numbers | config frozen |
| 10:30 | Run `sql/00_export_inputs.py` in Databricks: set the window, read the diagnostics cell, adjust the lever rules, run the rest | `inputs/` has `base_analise.parquet`, `base_analise_rules.json` + `sopi_quotas.csv`. The `captura_*` files are optional: until they exist, TAG / frequency / trade add no new levers in 2026/27 |
| 11:00 | `python run.py` on real data | `output/resultados.xlsx` + 11 charts |
| 13:00 | **Reconcile against notebook 07** (table below) | every gap explained by a config choice |
| 15:00 | Second scenario if needed: copy `config.yaml` → `cen2.yaml`, `python run.py --config cen2.yaml` | |
| 16:00 | Hand over: charts + Excel, note which `DECIDE` options were used | |

Runs where you prefer: on a laptop, or inside Databricks (put the folder in your Workspace,
then `%cd /Workspace/Users/<you>/protection_matrix` and `!python run.py`). The demo with
60k PDVs takes ~10 s.

## The 5 decisions (notebook 07 has two versions of each)

| config key | Default here | The other version in 07 |
|---|---|---|
| `simulation.levers` | all 6 levers | last cells: only aderido, sopi, vol_promo, tag |
| `simulation.cumulative` | `true` (2027 builds on 2026) | last cells: `false` (2027 vs today) |
| `simulation.vol_promo.band` | [0.55, 0.70) / [0.40, 0.55) | last cells: [0.60, 0.70) / [0.50, 0.60) |
| `capture.performance_curve` | −8.5% … +10% | last cells: −8.9% … +5.6% |
| `capture.months` / `reference_2027` / `base_2027` | 3 & 12 months, 2027 measured vs 2026, on vol_ly | Captura cell: vol_ref_26 × perf_27 (full level, not the gain) |

To reproduce the **last cells of 07** exactly: levers `[aderido, sopi, vol_promo, tag]`,
`cumulative: false`, bands `[0.60,0.70]` / `[0.50,0.60]`, the second curve, `months: {"26": 1, "27": 1}`,
`reference_2027: "baseline"`.

## What is fixed versus notebook 07 (on purpose)

- **G1 (PDM) / G2 (S5) keep their group** in 2026/27. In 07's final pivot they were reclassified as G3–G6, so the G1/G2 columns were always 0.
- **Capture matrices use SSS PDVs only**, the same population as every other chart.
- **Left joins only**: no PDV disappears when it is missing from a manual input file (it gets 0).
- **Draw order is a hash of seed + PDV**, so the same PDVs are drawn regardless of row order. It is not Spark's `rand`/`xxhash64`, so the exact PDVs drawn differ from 07 even though the counts per group × decile match.
- One `unb_pdv` format everywhere: `<unb>_<pdv>` without leading zeros.

## Reconciliation checklist (13:00)

| Check | Should match 07 exactly? |
|---|---|
| PDVs, SSS PDVs, % per lever | yes |
| Growth by # levers / by group / dynamic × group (point values) | yes |
| Bootstrap CIs | within ~0.2 pp (different random draws) |
| Aderido / SOPI counts per group × decile | yes (quotas are deterministic) |
| Which PDVs were drawn, migration, capture | no: different draw order + the fixes above |

## How base_analise is built (sql/00_export_inputs.py)

One row per PDV from the segmentation table, over an analysis window (default 2026-01 to 2026-06, LY = same months of 2025):

| Column | Rule | Confidence |
|---|---|---|
| PDV universe / `primary_dynamics` | segmentation PDVs with primary dynamic MERCADO, ATACADO or LC; secondary dynamic not filtered | your rule |
| `vol`, `vol_ly` | beer volume from `orders` with the notebook 03 filters | from notebooks |
| `wgt_promo_protect` | volume of SKUs on promo ≥ 7 days ÷ volume (notebooks 02 + 04) | from notebooks |
| `status_sss` | `vol > 0` and `vol_ly > 0` | **guess** |
| `status_sopi` | active SOPI cooler in the ref month (notebook 01) | from notebooks |
| `status_tag` | `tag_score = 'high'` in the ref month | likely (07 filtered "high") |
| `status_freq` | `current_frequency >= 2` in the ref month | **guess**: check the diagnostics cell |
| `status_vol_promo` | `wgt_promo_protect >= 0.70` | **guess**: band above the 2026/27 bands in 07 |
| `status_aderido` | PDVs in `aderido_source` | **no source**: 0 until you set one |
| `status_trade` | PDVs in `trade_source` | **no source**: 0 until you set one |
| `decil_potential` | from `potential_source` | **no source**: falls back to deciles of `vol_ly` |
| `grupo_exclusivo` | CT (segmentation, Score 5 contracts, national clients) → G1; Score 5 → G2; else by # levers | from notebooks |

Every rule is a parameter in the first cell. The assumptions actually used are printed and saved in `base_analise_rules.json` next to the parquet, so each run is traceable.

**Before trusting the results, fill `aderido_source` and `potential_source`.** Aderido is one of the six levers and the aderido draw only picks deciles 1–3, so the fallbacks change both the baseline and the simulation.

## Not covered today

- Notebook 06 (TTC) produced no output; its table is the likely `aderido_source` once you know which column marks adherence.
- Score RN (notebook 05) is commented out of the levers in 07, so it is not exported.
