# Shipment Visibility & Exception Platform: Research Report

> **Where the code is.** The experiment scripts that produced these results are part of the source package, which is distributed separately (see [SOURCE-ACCESS.md](../SOURCE-ACCESS.md)). The raw results they wrote are included in this repository under [`results/`](../results/).

> **Status: protocol v0.2, results recorded (final run of 2026-10-23).** Every number below was produced by the scripts in this repository (`uv run svp-experiments run`) and can be traced to `results/raw/*.json`, `results/summary.json` and `results/run_metadata.json`. **All data are synthetic**: they come from a seeded generator written by the same author as the models, there are no real carrier feeds or real shipment histories, and the "carrier-provided ETA" is itself simulated under a stated optimism model. High scores therefore measure self-consistency of a simulator and a model, not performance on real freight. Negative results are reported as negative.

## Abstract

The study asks whether a calibrated quantile-regression ETA model can outperform a (simulated) carrier-provided ETA and simple baselines, and how early it can flag shipments that will miss their promised time. On ten independent seeded synthetic worlds (about 18,800 backtested shipments in total), LightGBM quantile regression reduced the mean absolute error of the median remaining-time forecast by 58% against the simulated carrier ETA and by 22% against a lane-average baseline (H1 supported, the second margin narrowly). Its nominal 80% intervals under-covered (75.5%, with only 50% of lane and carrier groups inside the 75-85% band; H2 not supported); a split-conformal correction restored 79.9% coverage. The late-risk detector reached 80.2% precision and 69.8% recall with a median 57.5 hours of warning, just short of the 70% recall target (H3 not supported by the pre-stated rule). A canonical normaliser made 2.4% mapping errors against 44% for a naive pass-through of carrier codes and 10.7% for a per-carrier table without context rules (H4 supported; the pass-through baseline is deliberately naive and the vocabularies were written by the same author).

## 1. Research question

Can a calibrated quantile-regression ETA model outperform carrier-provided ETAs and simple baselines, and how early can it reliably flag shipments that will miss their delivery window?

## 2. Hypotheses

- H1: Gradient-boosted quantile regression reduces mean absolute ETA error by at least 20% versus the carrier-provided ETA and versus a lane-average baseline.
- H2: Predicted 80% intervals achieve empirical coverage between 75% and 85% across lanes and carriers.
- H3: The exception detector flags at least 70% of late shipments at least 12 hours before the promised date at 80% precision.
- H4: A canonical normalization layer reduces status-mapping errors by at least 50% versus passing through carrier codes.

**Decision rules fixed in v0.2 (before the final run).** For every hypothesis the replication unit is a seed (an independent synthetic world); the central value is the mean over seeds with a percentile-bootstrap 95% CI. Verdicts are three-level: *supported* if the CI clears the threshold, *inconclusive* if only the point estimate meets it, *not supported* otherwise. H1: the CI lower bound of the relative MAE reduction is at least 20% against both baselines. H2: the pooled 80% coverage CI lies inside [0.75, 0.85] and at least 80% of the 15 lane and carrier groups have coverage inside [0.75, 0.85]. H3: precision CI lower bound at least 0.80 and recall CI lower bound at least 0.70 for the detector with its threshold chosen on validation data. H4: the CI lower bound of the relative reduction in error rate versus pass-through is at least 50%.

## 3. Background

This section is a short positioning from memory; the citations in section 11 were **not re-checked against the original sources**.

- Quantile regression (Koenker and Bassett, 1978) estimates conditional quantiles by minimising the pinball loss, a strictly proper scoring rule for quantiles (Gneiting and Raftery, 2007). Gradient-boosted trees can optimise it directly (LightGBM, Ke et al., 2017).
- Independently fitted quantile models can cross; sorting them (rearrangement, Chernozhukov et al., 2010) is a simple repair and is what this project does.
- Quantile intervals fitted by boosting are often not calibrated out of sample; conformalised quantile regression (Romano et al., 2019) adds a correction from a held-out calibration set and gives finite-sample marginal coverage under exchangeability, which time-ordered data only approximately satisfy.
- Forecast evaluation for time-ordered data needs time-based splits (Hyndman and Athanasopoulos, 2021), which is why no random split across time is used here.
- Multiple-comparison control uses Holm's step-down procedure (Holm, 1979); intervals use the percentile bootstrap (Efron and Tibshirani, 1993).

## 4. Method

### 4.1 System under test

A logistics control tower prototype: carrier adapters (JSON webhook, X12 214-style text, saved emails) feed a lossless raw store and an idempotent normalising consumer; timelines and ETA features are derived from canonical events; a LightGBM quantile model forecasts remaining hours to delivery; a late-risk detector turns the predictive distribution into a probability of lateness. The experiments exercise the same code as the application (`svp.eta`, `svp.exceptions.detect`, `svp.adapters`, `svp.mapping`).

### 4.2 Environment: the synthetic generator

Data come only from `svp.synth` (seeded; parameters in `src/svp/synth/params.py`, copied into every run's config hash). Nothing is real.

- **World:** 14 locations and 12 lanes with 1 or 2 intermediate hubs (two lanes cross a border hub with customs), three carriers, two carriers per lane. City names are labels only.
- **Demand:** about 14 shipments a day for 255 days (weekday-weighted Poisson), about 3,460 shipments per seed.
- **Nominal schedule:** leg hours from great-circle distance at 45 km/h, planned hub dwell 6 h, promised time = booking + 12 h + 1.15 x nominal transit, rounded to 18:00 local at the destination and never on a Sunday.
- **Delay processes:** lognormal leg times with lane-specific mean multiplier (1.00-1.08) and sigma (0.08-0.20); gamma hub dwell; hubs dispatch at 06:00 and 18:00 local (queueing delay); a daily AR(1) congestion state per hub with a Monday bump; Poisson **weather shocks** (about 3 per 30 days, 300-900 km radius, 24-72 h, 0.2-0.5 extra hours per hour of overlap); customs holds (18% at the border hub, lognormal duration); failed first delivery attempts (5%); damage (2%); no Sunday deliveries.
- **Carrier-provided ETA (simulated optimism model).** At each reported event the carrier publishes `now + nominal remaining time x (1 - optimism) + 0.7 x any delay the carrier itself announced`, with optimism 8% (BRIGHTLINE), 15% (COBALT) and 3% (MERIDIAN). It ignores lane slowness, dispatch waits and unannounced shocks. It is optimistic by construction, so the size of the improvement over it depends on this assumption.
- **Reporting:** carrier-specific vocabularies and latencies (REST: minutes; EDI: 6-hourly batches; email: about 40 minutes), missed routine scans (1%, 4%, 8%), stuck messages (4% delayed 2-36 h, creating out-of-order arrival), byte-identical re-sends (5%), and, for the email carrier only, vocabulary drift (8% of eligible events use phrases the v1 mapping does not know).
- **Ground truth:** every emitted event has a true canonical status (used by E4) and every shipment has a true delivery time.

The defaults were fixed after looking at generator sanity statistics (late-delivery share, transit-to-nominal ratios) and before any model result; they were not tuned to make a hypothesis pass. In the final worlds 29.8% of backtested shipments were late (5,608 of 18,809).

### 4.3 Baselines

- **Carrier-provided ETA** (simulated, above): remaining time = latest published ETA minus now (floored at zero).
- **Lane-average transit time:** mean pickup-to-delivery hours per lane in the training window minus elapsed time, floored at 1 h; made into a distribution with the training window's empirical residual quantiles per lane.
- **Linear quantile regression:** one LP per quantile on standardised numeric features plus one-hot lane, carrier and stage (6,000-row random subsample of the training window).
- Additional context baselines declared in v0.2: *carrier ETA + empirical residual quantiles per carrier* and *lane x last-milestone average remaining time*.

### 4.4 Independent variables

- **Model family:** the baselines above and LightGBM quantile regression (300 trees, 15 leaves, learning rate 0.05, fixed a priori, no tuning, single thread). Quantile crossings are repaired by sorting.
- **Feature set:** full (17 features: lane, carrier, stage, elapsed time, schedule slippage at the last scan, carrier ETA, time to promise, customs and delay-notice flags, calendar, recent lane transit ratio, network and carrier hub-dwell ratios over 48 h), a variant without the carrier ETA, and a static-only variant (median only).
- **Prediction horizon:** reported by the known time to the promised date (under 24 h, 24-72 h, over 72 h).
- **Quantile level:** 0.05, 0.10, 0.25, 0.50, 0.75, 0.90, 0.95.

### 4.5 Metrics

MAE and MAPE of the median forecast (MAPE only on rows with at least 6 h remaining); mean pinball loss over the seven quantiles; empirical interval coverage (50%, 80%, 90%) overall, per lane and per carrier; detector precision, recall and lead time; status-mapping error rate. No timing metrics are used as outcomes.

### 4.6 Backtest design and leakage controls

Rows are (shipment, tick) pairs at every 00:00 and 12:00 on the simulation clock while the shipment is known to be in transit. Three expanding-window folds have test windows starting at day 120, 165 and 210 (45 days each). For a fold with test start C2 and a 30-day validation window starting C1: **train** = rows of shipments whose delivery had been *received* by C1; **validation** (calibration and threshold selection only, never used in a fit) = shipments first picked up in [C1, C2) and delivered by C2; **test** = all shipments first picked up in [C2, C2 + 45 days). Features at tick T use only events received by T and completed shipments received by T. Tests check this by truncating each shipment's history and by rewriting every event after a cutoff ("poisoning the future"), and assert the split invariants on a real backtest.

### 4.7 Compute environment

Windows 11 (as recorded: Windows-11-10.0.26200-SP0), AMD EPYC 7543P 32-core processor as exposed to this virtual machine (4 physical / 8 logical cores, 28.0 GB RAM), Python 3.12.13. The machine was shared with other builds and other applications: CPU utilisation sampled over one second was 90.4% (start), 90.1% (after stage a), 86.7% (after stage b), with 5.2-5.4 GB of RAM available. At most 2 worker processes ran at once, each single-threaded (LightGBM `n_jobs=1`, OpenMP/BLAS threads set to 1). The final invocation took 1,220 s. Run id 50f48e4f9b5d, config hash 68cfaf93b7280fd9, code hash 286e4d6f283d8a6a, lockfile hash ae41fd6685afe285.

## 5. Experiment matrix

| ID | Experiment | Manipulation | Measured |
|---|---|---|---|
| E1 | ETA accuracy | Time-based backtest of all models | MAE, pinball loss |
| E2 | Calibration | Interval coverage per lane and carrier | Empirical vs. nominal coverage |
| E3 | Early warning | Exception detector over rolling windows | Lead time, precision, recall |
| E4 | Normalization | Normalized vs. pass-through codes on labelled events | Mapping error rate |

Each experiment is a script (`src/svp/experiments/e1_...py` to `e4_...py`) and one resumable command runs all of them. Replications: 10 seeds (0-9), fixed before the final run.

## 6. Statistical plan

- **Replication:** 10 independent synthetic worlds (seeds 0-9). Rows within a shipment and shipments within a world are correlated, so the seed is the unit of replication; no within-seed confidence intervals are computed.
- **Summaries:** means over seeds with percentile-bootstrap 95% CIs of the mean (10,000 resamples, fixed seed). Per-seed raw values are in `results/raw`.
- **Comparisons:** paired exact two-sided Wilcoxon signed-rank tests across seeds (with 10 pairs the smallest attainable p-value is 0.002).
- **Effect sizes:** Cohen's d_z on the paired differences. These are very large because the per-seed differences are tightly clustered; they describe the simulator's noise across seeds and are not a claim about real-world effect sizes.
- **Multiple comparisons:** Holm-Bonferroni across the two H1 comparisons (against the carrier ETA and against the lane average). Every other comparison is secondary and exploratory and is not corrected.
- **No silent exclusions:** no seed, fold or row was excluded; the validation window drops shipments not delivered by its end (a selection effect on the validation data only, see section 9).
- **Negative results are results:** hypotheses H2 and H3 were not supported and are reported as such.

## 7. Results

All results: 10 seeds, 0 failed units, 0 retries, one uninterrupted invocation (see section 12). "n" counts backtest test rows (shipment x 12-hour tick) and shipments summed over the 10 seeds.

### E1. ETA accuracy

**Manipulation:** Time-based backtest of all models
**Measured:** MAE, pinball loss
**Data:** synthetic. The carrier-ETA rows use the simulated optimism model, not real carrier ETAs.

| Condition | n | Result | 95% CI | Notes |
|---|---|---|---|---|
| carrier_eta | 181,850 rows / 18,809 shipments | MAE 22.99 h | [22.55, 23.37] | median abs err 19.46 h; bias -21.60 h; MAPE 42.4%; pinball q50 11.49 h (median only) |
| carrier_eta_corrected | 181,850 rows / 18,809 shipments | MAE 15.17 h | [14.86, 15.46] | median abs err 12.70 h; bias -2.32 h; MAPE 37.0%; mean pinball 4.31 h |
| lane_average | 181,850 rows / 18,809 shipments | MAE 12.33 h | [12.01, 12.65] | median abs err 8.68 h; bias -1.38 h; MAPE 30.5%; mean pinball 3.59 h |
| lane_stage_average | 181,850 rows / 18,809 shipments | MAE 15.44 h | [15.14, 15.73] | median abs err 12.12 h; bias -1.71 h; MAPE 35.9%; mean pinball 4.44 h |
| lgbm | 181,850 rows / 18,809 shipments | MAE 9.60 h | [9.36, 9.84] | median abs err 5.29 h; bias -1.39 h; MAPE 20.7%; mean pinball 2.92 h |
| lgbm_no_carrier_eta | 181,850 rows / 18,809 shipments | MAE 9.57 h | [9.36, 9.80] | median abs err 4.95 h; bias -1.39 h; MAPE 20.7%; pinball q50 4.79 h (median only) |
| lgbm_static_only | 181,850 rows / 18,809 shipments | MAE 9.37 h | [9.15, 9.60] | median abs err 4.46 h; bias -1.27 h; MAPE 20.2%; pinball q50 4.68 h (median only) |
| linear_qr | 181,850 rows / 18,809 shipments | MAE 11.12 h | [10.88, 11.39] | median abs err 8.31 h; bias -1.68 h; MAPE 26.2%; mean pinball 3.29 h |

Comparisons of the LightGBM median forecast (paired across the 10 seeds; the first two rows are the H1 comparisons):

| Condition | n | Result | 95% CI | Notes |
|---|---|---|---|---|
| LightGBM vs carrier_eta | 10 seeds | MAE 58.2% lower | [57.6%, 58.8%] | Wilcoxon p 0.0020; Holm p 0.0039; d_z -30.3 |
| LightGBM vs carrier_eta_corrected | 10 seeds | MAE 36.7% lower | [36.0%, 37.3%] | Wilcoxon p 0.0020; exploratory; d_z -24.5 |
| LightGBM vs lane_average | 10 seeds | MAE 22.1% lower | [21.0%, 23.0%] | Wilcoxon p 0.0020; Holm p 0.0039; d_z -10.1 |
| LightGBM vs lane_stage_average | 10 seeds | MAE 37.8% lower | [36.5%, 39.2%] | Wilcoxon p 0.0020; exploratory; d_z -13.0 |
| LightGBM vs linear_qr | 10 seeds | MAE 13.7% lower | [12.7%, 14.5%] | Wilcoxon p 0.0020; exploratory; d_z -8.0 |

MAE (hours, mean over seeds) by known time to the promised date:

| Model | < 24 h | 24-72 h | > 72 h |
|---|---|---|---|
| carrier_eta | 11.8 | 20.4 | 33.0 |
| carrier_eta_corrected | 15.9 | 12.5 | 17.6 |
| lane_average | 10.5 | 12.5 | 13.3 |
| lane_stage_average | 11.8 | 14.9 | 18.4 |
| lgbm | 6.5 | 9.4 | 11.8 |
| lgbm_no_carrier_eta | 6.4 | 9.5 | 11.8 |
| lgbm_static_only | 6.2 | 9.3 | 11.6 |
| linear_qr | 9.1 | 10.9 | 12.7 |

### E2. Calibration

**Manipulation:** Interval coverage per lane and carrier
**Measured:** Empirical vs. nominal coverage (nominal 80% unless stated)
**Data:** synthetic.

| Condition | n | Result | 95% CI | Notes |
|---|---|---|---|---|
| carrier_eta_corrected | 181,850 rows | 80% coverage 0.803 | [0.797, 0.809] | mean width 47.7 h; groups in 75-85%: 79%; group range 0.723-0.872; 50% interval 0.499; 90% interval 0.897 |
| lane_average | 181,850 rows | 80% coverage 0.794 | [0.786, 0.802] | mean width 39.0 h; groups in 75-85%: 79%; group range 0.672-0.864; 50% interval 0.496; 90% interval 0.893 |
| lane_stage_average | 181,850 rows | 80% coverage 0.794 | [0.791, 0.798] | mean width 48.0 h; groups in 75-85%: 97%; group range 0.742-0.828; 50% interval 0.496; 90% interval 0.895 |
| lgbm | 181,850 rows | 80% coverage 0.755 | [0.749, 0.761] | mean width 29.4 h; groups in 75-85%: 50%; group range 0.682-0.801; 50% interval 0.468; 90% interval 0.860 |
| lgbm_cqr | 181,850 rows | 80% coverage 0.799 | [0.794, 0.805] | mean width 31.2 h; groups in 75-85%: 83%; group range 0.731-0.846 |
| linear_qr | 181,850 rows | 80% coverage 0.788 | [0.781, 0.795] | mean width 35.5 h; groups in 75-85%: 85%; group range 0.717-0.833; 50% interval 0.493; 90% interval 0.887 |

LightGBM nominal-80% coverage by group (mean over seeds, 95% CI over seeds):

| Group | Coverage | 95% CI |
|---|---|---|
| lane L01-LAX-NYC | 0.737 | [0.712, 0.759] |
| lane L02-SEA-BOS | 0.726 | [0.704, 0.747] |
| lane L03-SEA-DEN | 0.742 | [0.720, 0.765] |
| lane L04-LAX-CHI | 0.749 | [0.721, 0.773] |
| lane L05-DFW-MIA | 0.755 | [0.740, 0.770] |
| lane L06-CHI-ATL | 0.759 | [0.745, 0.774] |
| lane L07-DEN-NYC | 0.728 | [0.709, 0.748] |
| lane L08-MSP-LAX | 0.756 | [0.727, 0.784] |
| lane L09-NYC-MIA | 0.758 | [0.738, 0.778] |
| lane L10-ATL-SEA | 0.765 | [0.751, 0.781] |
| lane L11-DFW-MEX | 0.753 | [0.724, 0.779] |
| lane L12-LAX-MEX | 0.737 | [0.720, 0.756] |
| carrier BRIGHTLINE | 0.756 | [0.749, 0.763] |
| carrier COBALT | 0.754 | [0.744, 0.763] |
| carrier MERIDIAN | 0.753 | [0.741, 0.765] |
| time to promise 24_72h | 0.761 | [0.755, 0.767] |
| time to promise gt72h | 0.756 | [0.748, 0.764] |
| time to promise lt24h | 0.743 | [0.736, 0.749] |

### E3. Early warning

**Manipulation:** Exception detector over rolling windows
**Measured:** Lead time, precision, recall (alert at least 12 h before the promised time; late = delivered after the promised time)
**Data:** synthetic. `@selected` thresholds are chosen on each fold's validation window (smallest tau with validation precision at least 0.80); `@0.5` is a fixed threshold; the `rule_` detectors have no threshold.

| Condition | n | Result | 95% CI | Notes |
|---|---|---|---|---|
| carrier_eta_corrected@0.5 | 18,809 shipments (5,608 late) | precision 0.357 / recall 0.982 | P [0.343, 0.371]; R [0.977, 0.986] | median lead 67.0 h; 15,418 alerted, 5,505 true alerts |
| carrier_eta_corrected@selected | 18,809 shipments (5,608 late) | precision 0.755 / recall 0.317 | P [0.731, 0.779]; R [0.234, 0.396] | median lead 35.0 h; 2,366 alerted, 1,799 true alerts; mean tau 0.93 |
| lgbm@0.5 | 18,809 shipments (5,608 late) | precision 0.500 / recall 0.933 | P [0.484, 0.513]; R [0.925, 0.941] | median lead 82.4 h; 10,485 alerted, 5,237 true alerts |
| lgbm@selected | 18,809 shipments (5,608 late) | precision 0.802 / recall 0.698 | P [0.785, 0.819]; R [0.674, 0.723] | median lead 57.5 h; 4,896 alerted, 3,922 true alerts; mean tau 0.83 |
| linear_qr@0.5 | 18,809 shipments (5,608 late) | precision 0.416 / recall 0.959 | P [0.403, 0.429]; R [0.954, 0.963] | median lead 85.4 h; 12,917 alerted, 5,378 true alerts |
| linear_qr@selected | 18,809 shipments (5,608 late) | precision 0.803 / recall 0.706 | P [0.785, 0.823]; R [0.676, 0.735] | median lead 51.3 h; 4,955 alerted, 3,967 true alerts; mean tau 0.87 |
| rule_carrier_eta | 18,809 shipments (5,608 late) | precision 0.763 / recall 0.604 | P [0.745, 0.781]; R [0.588, 0.622] | median lead 35.0 h; 4,440 alerted, 3,393 true alerts |
| rule_lane_average | 18,809 shipments (5,608 late) | precision 0.527 / recall 0.824 | P [0.513, 0.543]; R [0.807, 0.837] | median lead 88.2 h; 8,761 alerted, 4,622 true alerts |

Operating curve of the LightGBM detector on pooled test data (mean over seeds; descriptive, nothing is selected on it):

| tau | Precision | Recall | Median lead (h) |
|---|---|---|---|
| 0.1 | 0.309 | 0.997 | 95.9 |
| 0.2 | 0.331 | 0.989 | 95.0 |
| 0.3 | 0.372 | 0.973 | 87.4 |
| 0.4 | 0.427 | 0.955 | 86.6 |
| 0.5 | 0.500 | 0.933 | 82.4 |
| 0.6 | 0.575 | 0.906 | 77.5 |
| 0.7 | 0.663 | 0.848 | 74.0 |
| 0.8 | 0.769 | 0.761 | 62.5 |
| 0.9 | 0.877 | 0.605 | 49.0 |

### E4. Normalization

**Manipulation:** Normalized vs. pass-through codes on labelled events
**Measured:** Mapping error rate (UNKNOWN counts as an error)
**Data:** synthetic payloads (JSON, X12 214 text, .eml) parsed by the real adapters; ground-truth labels come from the generator.

| Condition | n | Result | 95% CI | Notes |
|---|---|---|---|---|
| ALL: normaliser v1 | 323,433 events | error 2.368% | [2.244%, 2.487%] | reduction vs pass-through 94.7% [94.5%, 94.8%]; vs per-carrier table 77.6% [75.5%, 79.4%] |
| ALL: per-carrier table (no context rules) | 323,433 events | error 10.651% | [10.156%, 11.118%] |  |
| ALL: pass-through (shared lexicon) | 323,433 events | error 44.280% | [42.582%, 45.750%] |  |
| BRIGHTLINE: normaliser v1 | 106,189 events | error 0.000% | [0.000%, 0.000%] | reduction undefined (baseline error is 0) |
| BRIGHTLINE: per-carrier table (no context rules) | 106,189 events | error 0.000% | [0.000%, 0.000%] |  |
| BRIGHTLINE: pass-through (shared lexicon) | 106,189 events | error 0.000% | [0.000%, 0.000%] |  |
| COBALT: normaliser v1 | 108,087 events | error 0.000% | [0.000%, 0.000%] | reduction vs pass-through 100.0% [100.0%, 100.0%]; vs per-carrier table 100.0% [100.0%, 100.0%] |
| COBALT: per-carrier table (no context rules) | 108,087 events | error 24.769% | [24.498%, 25.029%] |  |
| COBALT: pass-through (shared lexicon) | 108,087 events | error 31.459% | [31.020%, 31.820%] |  |
| MERIDIAN: normaliser v1 | 109,157 events | error 7.022% | [6.857%, 7.161%] | reduction vs pass-through 93.0% [92.8%, 93.1%]; vs per-carrier table 0.0% [0.0%, 0.0%] |
| MERIDIAN: per-carrier table (no context rules) | 109,157 events | error 7.022% | [6.857%, 7.161%] |  |
| MERIDIAN: pass-through (shared lexicon) | 109,157 events | error 100.000% | [100.000%, 100.000%] |  |

All 7,654 normaliser errors are Meridian events whose phrasing was injected as vocabulary drift and mapped to UNKNOWN by mapping v1 (the confusion table is in `results/summary.json`). The BRIGHTLINE and COBALT 0% errors are self-consistency by construction: the generator and the mapping files were written from the same vocabulary table by the same author. Mapping v2 (which covers the drift phrases) is used to demonstrate re-normalisation of history and is not scored here.

## 8. Discussion

**H1 (supported, with a caveat about the margin).** The LightGBM median forecast cut MAE from 23.0 h (simulated carrier ETA) and 12.3 h (lane average) to 9.6 h. The 58% gain over the carrier ETA is mostly a statement about the simulator's optimism model: the carrier ETA ignores lane slowness, dispatch waits and shocks by construction, and even adding the training window's residual quantiles to it (a free recalibration) still leaves the model's MAE 37% lower. The 22% gain over the lane average is the more informative number and its CI lower bound (21.0%) clears the 20% bar by one percentage point; one of ten seeds fell below 20% (19.0%). Two further results temper the story. The static-only feature set (no carrier ETA, no schedule slippage, no recent-history context) was slightly *better* than the full model (MAE 9.37 h versus 9.60 h), so in this simulator the engineered history features did not help and may add variance; and LightGBM's MAE was only 13.7% lower than a plain linear quantile regression's. The time-to-promise breakdown shows the absolute and relative advantage over the carrier ETA is largest far from delivery (MAE 33.0 h versus 11.8 h beyond 72 h) and smaller in the last day (11.8 h versus 6.5 h).

**H2 (not supported).** The LightGBM 80% intervals covered 75.5% [74.9, 76.1] of outcomes and only 50% of the lane and carrier groups fell inside 75-85%; the lowest lane coverage was 72.6% (SEA-BOS) and the highest 76.5% (ATL-SEA). The 50% and 90% intervals under-cover too (46.8% and 86.0%): the intervals are too narrow. Boosted quantile models with fixed hyperparameters fitted on one time window are expected to be over-confident when the next window has different shocks; no hyperparameters were tuned to repair this. Cheap alternatives were better calibrated: linear quantile regression (78.8%), the lane average with empirical residual quantiles (79.4%) and a conformalised version of the LightGBM interval (79.9% [79.4, 80.5], 83% of groups inside the band, at the cost of 6% wider intervals, 31.2 h versus 29.4 h). Conformalisation assumes exchangeability that time-ordered data only approximately satisfy; here it worked, in a world whose generator is stationary apart from random shocks.

**H3 (not supported, narrowly).** With the threshold chosen on validation data (mean tau 0.83; the 80% precision target was reached on every validation window), the detector reached precision 0.802 [0.785, 0.819] and recall 0.698 [0.674, 0.723]. Precision meets the target; recall is 0.2 percentage points below the 70% bar and its CI includes values well below it, so the pre-stated rule says not supported. The median warning was 57.5 hours. A rule that alerts when the carrier's own ETA is later than the promise had precision 0.763 and recall 0.604 with only 35 hours of median lead, so the model adds about 9 points of recall at a similar precision and much more lead time; but a linear quantile model reached the same operating point (0.803 / 0.706), so the gain comes from calibrated predictive distributions rather than from boosting. The operating curve shows the trade-off: at tau 0.8 precision is 0.77 and recall 0.76.

**H4 (supported, with two caveats).** The normaliser's overall error was 2.4% against 44.3% for pass-through (94.7% reduction). That baseline shares one code lexicon across carriers, so it scores 0% error on the carrier whose vocabulary it copies and 100% on the email carrier; it is a straw man by design. The more realistic comparison is a per-carrier code table without location context: the normaliser reduced its error by 77.6% [75.5, 79.4], entirely through the EDI context rules (the per-carrier table is wrong on 24.8% of Cobalt events). The remaining 2.4% of errors are all vocabulary drift on the email carrier, which a mapping update fixes (demonstrated by re-normalising history from raw payloads); the sensitivity of E4 to the injected drift rate (8%) was not explored.

**What the results do and do not show.** They show that the code behaves as designed on data whose statistical structure the author chose: leakage controls hold, calibration can be measured and repaired, and a probability-of-lateness detector beats a rule on the carrier's ETA. They do not show how optimistic real carrier ETAs are, how real vocabularies drift, or whether any effect size would survive contact with real freight.

## 9. Threats to validity

- **Synthetic data and shared authorship.** Generator, models, vocabularies and mappings were written by the same author; the results measure self-consistency. Real delay processes are likely heavier-tailed, more heterogeneous and less stationary. Nothing here is evidence about real carriers.
- **Assumed carrier-ETA optimism.** The carrier-ETA baseline is simulated under a stated optimism model; the H1 margin against it is a function of that assumption.
- **Weak baseline for H4.** The pass-through baseline is naive by construction; the per-carrier table is the fairer comparison.
- **Distribution shift:** the generator is stationary except for random shocks; rolling re-evaluation was done across three expanding windows but no structural change (new carriers, or vocabulary changes after launch other than the injected drift) was tested.
- **Label leakage:** features are restricted to events received at prediction time and checked by truncation and poisoning tests; row selection (a row exists while the delivery has not yet been received) uses received times only. The validation window excludes shipments not yet delivered by its end, which biases validation toward faster shipments and may lower the selected threshold slightly.
- **Statistical dependence:** rows within a shipment, shipments within a lane and folds within a world are dependent; CIs are over 10 worlds only and describe variation across simulator seeds, not across real-world conditions.
- **Researcher degrees of freedom:** the analysis choices are fixed in this protocol; deviations are logged in the change log. Development smoke runs used a reduced configuration and a development seed (9001) and were used to find bugs; no generator parameter, hyperparameter or decision rule was changed afterwards, but the author had seen those preliminary numbers before freezing the code.
- **Fixed hyperparameters.** LightGBM settings were not tuned, so H2 and the static-only result may differ with tuning.
- **Machine contention:** the experiments ran on a shared machine that was 87-90% CPU-busy; the metrics are not timing-based and each unit is single-threaded and seeded, so contention should affect wall time and not results, but this was not verified by a repeat run.
- **Multiple comparisons:** only the two H1 tests are corrected; the other comparisons in the tables are exploratory.

## 10. Reproducibility checklist

- [ ] Code tagged at the commit used for the reported results (the working directory was never put under version control; the run records a hash of the code that determines results instead: `code_hash` in `results/run_metadata.json`)
- [x] Environment pinned: `uv.lock` (its hash is in the run metadata); no container image was used, so there is no image digest
- [x] Random seeds and run counts recorded (seeds 0-9; 50 units; config hash in every raw result)
- [x] Raw results published: per-seed JSON in `results/raw`, aggregated by `results/summary.json` and `results/tables` (an aggregation script, not a notebook)
- [x] Hardware or service configuration described in full (section 4.7 and `results/run_metadata.json`)
- [x] One command reproduces every table: `uv run svp-experiments run` (no figures are produced)
- [x] Data sources, licenses and versions listed: no external data; the generator and its parameters are in the repository and the parameter digest is in the config hash

## 11. References

- Koenker, R. & Bassett, G. (1978). Regression Quantiles. Econometrica 46(1).
- Hyndman, R. & Athanasopoulos, G. (2021). Forecasting: Principles and Practice (3rd ed.). OTexts.
- Gneiting, T. & Raftery, A. E. (2007). Strictly Proper Scoring Rules, Prediction, and Estimation. Journal of the American Statistical Association 102(477).
- Romano, Y., Patterson, E. & Candes, E. (2019). Conformalized Quantile Regression. Advances in Neural Information Processing Systems 32.
- Ke, G. et al. (2017). LightGBM: A Highly Efficient Gradient Boosting Decision Tree. Advances in Neural Information Processing Systems 30.
- Chernozhukov, V., Fernandez-Val, I. & Galichon, A. (2010). Quantile and Probability Curves Without Crossing. Econometrica 78(3).
- Holm, S. (1979). A Simple Sequentially Rejective Multiple Test Procedure. Scandinavian Journal of Statistics 6(2).
- Efron, B. & Tibshirani, R. (1993). An Introduction to the Bootstrap. Chapman & Hall.

> These citations were written from memory and were not re-verified against the original sources.

## 12. Change log

| Date | Version | Change |
|---|---|---|
| 2026-10-21 | v0.1 | Protocol drafted; no results yet. |
| 2026-10-23 | v0.2 | Protocol made executable before the final run: fixed the generator and its stated carrier-ETA optimism model; fixed 10 seeds, a three-fold expanding-window backtest with a validation window, and the three-level decision rules in section 2; defined the lane-average baseline; added the declared secondary analyses (carrier ETA with residual quantiles, lane x stage average, feature-set ablations, split-conformal intervals, a per-carrier code table baseline for E4); defined alerts and precision/recall at shipment level for E3. |
| 2026-10-23 | v0.2 | Development runs before freezing: a full-configuration backtest on development seed 9001 and two runs of the reduced "quick" configuration (2 seeds, 90 days) through the runner. They were used to find and fix bugs (an SSL environment crash in the webhook test, thread handling in the API, formatting) and are not kept or reported. No generator parameter, hyperparameter or decision rule was changed after seeing their numbers, but the author did see them. |
| 2026-10-23 | v0.2 | **Final run:** one uninterrupted invocation of `svp-experiments run` (started 16:25 UTC, 1,220 s, 50 units, 0 retries, 0 failures, 0 units resumed). No earlier partial run exists in `results/`; no rerun was made; the results above are from this run. |
| 2026-10-23 | v0.2 | H2 and H3 were not supported and were not re-run or re-tuned. |

---

[README](../README.md) · [Client case study](case-study.md) · [Research report](research-report.md)
