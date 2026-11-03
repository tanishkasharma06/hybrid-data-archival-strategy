# Hybrid Data Archival Strategy: A Comparative Study

A multi-factor archival scoring model (storage size, recency, access frequency) compared against a conventional time-based rule, implemented in SQL on Oracle Database over a synthetic IoT sensor dataset (100,000 readings, 3.5M access events).

**Result:** at a comparable archive volume (~18K records each), the hybrid model captured **95.23%** of cold records vs **88.32%** for the time-based rule (+6.91 percentage points, ~7.8% relative), and archived **73.33% fewer actively-used records** (556 vs 2,085).

| Metric | Time-based (>= 230 days) | Hybrid (H >= 2.5) |
|---|---|---|
| Records archived | 18,352 | 18,095 |
| Cold records archived | 16,267 | 17,539 |
| Active records archived | 2,085 | 556 |
| Cold capture rate | 88.32% | 95.23% |

## Problem

Time-based archival ("archive anything not touched in N days") ignores how often a record was used recently and how much space it occupies. Two records of the same age can deserve different decisions, and archiving a record that is still being read adds retrieval overhead from slow storage.

## Approach

For each record *i*:

- `U_i` = record size / total database capacity
- `D_i` = days since last access
- `f_i` = accesses in the last 100 days / 100

**Hybrid score:** `H_i = U_i * ln(D_i + 1) / (f_i + 0.01)`

Larger and staler records score higher; frequent recent access pushes the score down. Records with `H_i >= 2.5` are archived. The time-based baseline archives records with `D_i >= 230`. Both thresholds were set so the two strategies archive a similar number of records, so the comparison is about **selection quality, not volume**.

**Evaluation metrics:** records archived, cold records archived (0 accesses in last 100 days), active records archived, cold capture rate, reduction in active records archived.

## Repository structure

```
.
├── README.md
├── report/
│   └── Hybrid_Archival_Strategy_Report.pdf
├── notebooks/
│   └── 01_synthetic_data_generation.ipynb   # Pandas/NumPy data generation (Colab)
├── sql/
│   ├── 01_v1_prototype_global_utilisation.sql
│   ├── 02_per_record_score_and_threshold_calibration.sql
│   ├── 03_final_decision_view.sql
│   └── 04_strategy_comparison.sql
└── data/                                    # sample CSVs (optional)
```

## How to reproduce

1. Run the notebook to generate `SENSOR_READING` and `DATA_ACCESS_LOG` (CSV export).
2. Load both tables into Oracle and create a `STORAGE_CONFIGURATION` table holding `TOTAL_CAPACITY_MB`.
3. Run the SQL files in order (02 creates `HYBRID_ARCHIVAL_NEW`, which 03 and 04 depend on).

Schema:
- `SENSOR_READING(READING_ID, SENSOR_ID, READING_VALUE, READING_TIMESTAMP, DATA_SIZE, LAST_ACCESSED)`
- `DATA_ACCESS_LOG(ACCESS_ID, READING_ID, ACCESS_TIMESTAMP)`

## How the SQL evolved

The four scripts reflect real iteration, not one clean pass:

- **01:** first prototype used a single global utilisation constant, so size had no per-record effect. Threshold of 250 was explored.
- **02:** switched to per-record utilisation, which changed the score scale, then calibrated the threshold by ranking scores against the target archive volume.
- **03:** froze the final threshold (2.5) into a decision view.
- **04:** side-by-side comparison of both strategies.

## Limitations

- **Synthetic data**: access patterns were shaped by me, so absolute numbers will not transfer to a real workload.
- **Circularity risk**: "cold" is defined as zero accesses in 100 days, and the hybrid score uses that same 100-day access frequency while the baseline does not. The hybrid model therefore has a built-in advantage on this metric.
- **Threshold tuned on the evaluation data**: no train/test split was used.
- **Archival was simulated**; no real storage tiers or retrieval costs were measured.

## Future work

- Evaluate with a held-out time window (calibrate on past access, test on future access) so "cold" is not defined by the model's own input.
- Compare against stronger baselines (LRU/LFU-style, size-aware rules).
- Sensitivity analysis on the 0.01 smoothing constant, the 100-day window and the thresholds.
- Replace the hand-set formula with a learned model.

## Tech

Oracle SQL (views, CTEs, window functions), Python (Pandas, NumPy), Google Colab.

## Author

Tanishka Sharma, B.Tech CSE, Vellore Institute of Technology
