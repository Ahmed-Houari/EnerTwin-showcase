# EnerTwin v3: the all-season surrogate v2 against a simple fixed rule (2026-10-18)

Milestone (owner, 2026-10-18): the AI must never do worse than a simple fixed rule, and must understand all seasons;
the validated June results stay reproducible. Every figure is a model result: simulated control of a simulated building
(EnergyPlus v3), Riyadh TMYx weather, simulated occupancy. No meter, sensor or equipment is involved.

**Single source of truth for the numbers:** `docs/v3_results.md`. This document records the method and the evidence.

## Update 2026-10-19: the owner options and a fourth, untouched test (section 8)

With the owner's two options (a: a 26.0 °C hold only with a projected warmest occupied zone ≤ 25.9 °C; b: the rule's
25.5 °C hold also below the benefit floor when it still saves, keeps comfort and the peak) the manual AI matches the
fixed rule in December, June and March and trails it by 1.0 kWh on the new, untouched test (2–15 February, held out of
a refit). Flexible AUTO matches rule + lighting in June and March, trails by 0.9 kWh in February and by 61.4 kWh in
December, where a below-floor rule hold tripped the early-read guard on a day warmer than forecast. With the options the
AI is in practice the fixed rule (three different choices in 40 workdays). "Never worse" is still not met exactly.

## Results in brief (2026-10-18, v2 without the options)

**The milestone is not met everywhere.** The all-season v2 matches the fixed rule in June, beats it with manual approvals
in March (by 4.9 kWh, with 10 min of one zone at 26.000013 °C), and trails it in the new winter test (by 4.9 kWh manual,
4.6 kWh Flexible AUTO) and with Flexible AUTO in March (by 51.4 kWh, after the comfort guard suspended AUTO). It
understands the seasons much better than v1 (fallback on 1 of 10 winter workdays instead of 10, a different setpoint by
season, no action on cold days), and every one of its 50 decisions where the rule's own projection was beneficial chose an
action EnergyPlus projected at least as good as the rule. What separates "projected never worse" from "verified never
worse" is shown below: the benefit floor on cold days, and the 26 °C hold sitting exactly on the comfort bound.

Energy used over each fortnight (lower is better; "below no action" is the robust comparison, as in March):

| Period | No action | v2 AI manual | Fixed rule | v2 Flexible AUTO (0 approvals) | Fixed rule + lighting |
|---|---|---|---|---|---|
| **14–27 Dec (primary, untouched)** | 6,702.4 | 6,657.9 (44.5 below) | **6,653.0 (49.4 below)** | 6,608.5 (93.9 below) | **6,603.9 (98.5 below)** |
| 8–21 Jun (validated) | 15,678.3 | 14,156.9 (1,521.4 below) | 14,156.9 (1,521.4 below) | 14,089.2 (1,589.1 below) | 14,089.2 (1,589.1 below) |
| 2–15 Mar (seen once) | 7,116.2 | **6,991.6 (124.6 below)** | 6,996.5 (119.7 below) | 6,997.2 (119.0 below) | **6,945.8 (170.4 below)** |

| Period | AI manual against the rule | Flexible AUTO against rule + lighting |
|---|---|---|
| December | **trails by 4.9 kWh** (0.07 % of no action) | **trails by 4.6 kWh** |
| June | matches exactly (verification for verification) | matches exactly |
| March | **beats by 4.9 kWh**, but 0.17 occupied zone-hours above 26.0 °C | **trails by 51.4 kWh** (comfort guard suspended AUTO on Tue 13; Wed–Thu expired) |

## 1. Test sets locked first

Locked in `config/v3_test_periods.json` and enforced by `scripts/v3_test_lock.py` **before any v2 data, sizing run or
artifact existed** (git commit `21b6c43`, the first commit of the milestone; the data generator and the trainer refuse
anything that breaks the lock):

| Test | Days | Role | Strict |
|---|---|---|---|
| `winter_fortnight` | Fri 14 – Thu 27 December | **primary, new and untouched** | yes: no training run simulates any of its days, lead-in included |
| `summer_fortnight` | Fri 8 – Thu 21 June | the validated v1 held-out test | no |
| `spring_fortnight` | Fri 2 – Thu 15 March | second test, **already seen once** (the v1 surrogate fell back on all ten workdays; that result motivated this milestone) | no |

Rules: no training row's issue day, forecast day (the day before: its hours are the persistence forecast) or horizon day
is a test day; no training period's reported days include a test day; for the strict winter test no training run touches
any of its days. For June and March a training run may pass through their days only inside its 14-day no-action warm-up
(`train_march` warms up on 2–15 March, `train_june_late` on 8–21 June); no row, feature day or target is taken there.

**Why 14–27 December** (chosen on weather and feasibility only, before any model or rule result):

- Winter: a different season from June (summer) and March (spring). Two whole Fri–Thu weeks, like the other two.
- All of December in the TMYx file comes from one source year (2016): 14 consecutive real days, no month splice. Only
  the lead-in (from 30 November) crosses the Nov/Dec seam, as June's and March's lead-ins do.
- Mixed winter weather: week 1 cold (workday daily max 16.0–21.9 °C), week 2 milder (19.4–29.0 °C), so the test asks
  both "act" and "do not act".
- The coldest Fri–Thu fortnight that still leaves training days in its own month while no training run touches it.
  7–20 December would leave December one training day; any January fortnight none (a period cannot start before
  15 January: its 14-day lead-in would start in 2000); 2–15 or 9–22 February would leave February almost none (the March
  test's lead-in starts 16 February).

**Sizing and acceptance** (`python -m scripts.energy_model_v3_occupancy --period winter_fortnight`, the validated method):
HVAC sizes identical to the reference (the same design days), 0 unmet occupied cooling hours, lighting error 0.0 kWh,
occupant-hour error 0.003 h. No-action fortnight (occ_room): **6,702.4 kWh**, of which cooling 663.0 kWh. The stepping
runtime matches the batch on all 4,032 steps (zone air max diff 8.3×10⁻⁶ °C, facility 4.3 J;
`tmp/v3_allseason/winter/shadow/runs/2001-12-14_room_14d/verification.json`, status pass).

## 2. Candidates: the rule is always confirmed (optimizer `surrogate_v2`)

The v2 optimizer has EnergyPlus project, at **every workday decision**, the fixed rule and the permission's warmest hold
(`scripts/v3_surrogate.rule_actions`):

- `hold_25.5_0900_1800`: the fixed rule, cooling 25.5 °C from the issue step (09:00) to 18:00;
- `hold_26.0_0900_1800`: the permission's warmest full-day hold;
- `lighting_delay_5`: lights off 5 min after a room empties (the rule + lighting variant's lighting action).

They are grid actions (same ids, titles and schedule changes), inside the Flexible permission's envelope. When the
surrogate screens, they join its top 3 and its best lighting estimate (`screen(..., must=...)`); when it falls back
(input outside the training range, no cooling estimate passing, a model error), they join the detection list
(`cf.evaluate(..., always=...)`), so **even without the surrogate** the rule's own action is projected.

**The choice** (`v3_demo_runtime._recommend_v2`): the largest projected saving among the candidates that keep comfort,
pass the optimizer's energy thresholds (≥ 2 kWh and 0.5 %) and do not raise the day peak (the gate `beneficial` and AUTO
use). Only when none qualifies, the v1 choice. Under the Flexible permission the choice is narrowed to the candidates
the envelope allows, when one qualifies (the rule's hold always does), so an out-of-scope pick never leaves AUTO idle.
Consequence (unit-tested over 3,000 random decisions): **whenever the rule's own projection is beneficial, the chosen
action is projected to save at least as much as the rule.** When the rule's projection is below the thresholds (cold
days), the AI takes no action by design (the benefit floor is a pre-execution check of the permission), while the
fixed rule still acts.

What this guarantee does not cover: a projection uses the persistence forecast (yesterday's weather); the verified saving
uses the real weather. A pick projected better than the rule can verify worse. Section 5 measures it.

Unchanged: every guard (EnergyPlus confirmation, the pre-execution checks, the live comfort guard, the early read, the
verification suspension, revoke, expiry, audit, exactly-once, exact replay), the envelope bounds (`sg.ENVELOPE`,
`sg.in_envelope`), `AUTO_SCOPES`. Nothing widened.

## 3. The validated June path is pinned behind its flags

`--optimizer energyplus` (default) and `--optimizer surrogate` are the validated optimizers, unchanged: `surrogate` loads
the v1 artifact (`data/surrogate_v3/model/v3_surrogate_v1.joblib`, sha256 `5d40ce76…`, prediction fingerprint `45d8d6dc…`,
both pins unchanged), screens with the v1 rule (top 3, the exact hold under that permission, the best lighting) and
chooses with the v1 `_recommend`. The v2 path is `--optimizer surrogate_v2` with its own artifact and pins
(`data/surrogate_v3/model_v2/v3_surrogate_v2.joblib`). A run pins its optimizer and artifact in its manifest, so neither
can resume the other.

## 4. Training data: all twelve months, the locked tests excluded

Seven training periods were added to the six of v1 (`v3.PERIODS`, each inside one TMYx source month, each passing the
lock), so every month has training days. Same method as v1 (`python -m scripts.v3_surrogate_data --v2 --workers 20`):
the validated sizing per period, two persisted context runs (no action; one random grid action per workday), at 09:00
of every workday whose horizon is complete inside the period a seeded random 40 of the 242 options plus the baseline,
one anchor day per period with the whole grid, a third of the days also at 11:00 or 13:00; all EnergyPlus, through the
optimizer's own `cf.simulate`; seed 20261017.

| Period | Days | Sample days | Rows | No-action electricity / cooling (occ_room batch) | HVAC sizes identical | Unmet occupied cooling h |
|---|---|---|---|---|---|---|
| `train_january` (15–31 Jan) | 17 | 10 | 1,284 | 8,604.3 / 740.8 kWh | yes | 0.00 |
| `train_february` (1–28 Feb) | 28 | 17 | 1,884 | 13,836.0 / 1,390.0 kWh | yes | 0.00 |
| `train_march` (16–31 Mar) | 16 | 7 | 1,024 | 7,900.1 / 1,234.4 kWh | yes | 0.00 |
| `train_june_early` (1–7 Jun) | 7 | 2 | 604 | 7,682.7 / 3,804.1 kWh | yes | 0.00 |
| `train_june_late` (22–30 Jun) | 9 | 2 | 584 | 8,329.7 / 3,919.3 kWh | yes | 0.00 |
| `train_november` (1–30 Nov) | 30 | 18 | 2,004 | 16,293.2 / 3,082.7 kWh | yes | 0.00 |
| `train_december` (1–13 Dec) | 13 | 7 | 1,044 | 6,568.3 / 808.5 kWh | yes | 0.00 |

Occupant-hour error ≤ 0.0061 h and lighting error 0.0 kWh in every period. **v2 additions:** 154 decisions, 8,582
EnergyPlus runs, **8,428 rows** (`data/surrogate_v3/v2/dataset.jsonl`, sha256 `ff9a9940…`, manifest with the lock's
sha256 `b978754f…`), 128 min on 20 workers (2.29 s of wall time per run per decision); every row's replay of the persisted
past was exact (max diff 0.0), no duplicate. Together with the v1 dataset (11,664 rows, sha256 `59594cd3…`, checked):
**20,092 rows, 168 calendar days, all twelve months.** The coldest stretch of the year, 14 December – 14 January, has no
training day (the winter test, and no lead-in can start before 1 January), so the winter test sits at the edge of the
training data, not inside a month the model saw densely.

Running it on this laptop (32 threads, 31.2 GB RAM, commit limit 43.7 GB): 20 recycled workers (`max_tasks_per_child=4`,
no numpy in workers), at the same time as up to two fortnight runs and an 8-worker truth grid. Free RAM never fell below
3.0 GB and commit peaked at 34.6 of 43.7 GB (`tmp/v3_allseason/logs/resources.log`, one line a minute); no job failed.

## 5. Model choice (same honest method)

`python -m scripts.v3_surrogate_train --version v2 --jobs 16`: the six model families of v1 per target, compared by
day-grouped 5-fold cross-validation (the pick: lowest MAE under 1 ms per candidate), and beside it a **leave-one-month-
out** cross-validation (each month predicted by a model that never saw it).

| Target: MAE (R²) | Constant | Ridge | kNN | HGBR | Extra trees | MLP |
|---|---|---|---|---|---|---|
| kWh change, day-grouped | 32.57 (−0.01) | 13.94 (0.80) | 6.29 (0.95) | **2.26 (0.994)** | 2.49 (0.992) | 2.37 (0.994) |
| kWh change, leave-one-month-out | 34.22 (−0.10) | 15.09 (0.77) | 9.24 (0.90) | 3.00 (0.989) | 4.05 (0.979) | 3.02 (0.990) |
| Peak change kW, day-grouped | 9.23 (0.00) | 5.40 (0.66) | 3.49 (0.82) | 2.13 (0.93) | **1.83 (0.94)** | 1.92 (0.95) |
| Peak change kW, leave-one-month-out | 9.39 (−0.04) | 5.75 (0.62) | 4.15 (0.76) | 2.76 (0.90) | 2.39 (0.92) | 3.45 (0.59) |
| Outside-comfort hours, day-grouped | 2.07 (0.00) | 2.09 (0.29) | 0.87 (0.63) | 0.48 (0.84) | **0.48 (0.84)** | 0.64 (0.76) |
| Outside-comfort hours, leave-one-month-out | 2.08 (−0.01) | 2.14 (0.26) | 0.89 (0.60) | 0.53 (0.80) | 0.51 (0.82) | 0.73 (0.76) |
| Max occupied air °C, day-grouped | 0.58 (−0.02) | 0.15 (0.87) | 0.15 (0.87) | 0.08 (0.82) | **0.07 (0.85)** | 0.56 (−1.69) |
| Max occupied air °C, leave-one-month-out | 0.58 (−0.04) | 0.20 (0.71) | 0.18 (0.80) | 0.13 (0.70) | 0.10 (0.78) | 1.32 (−25.1) |

Choice (as v1): gradient boosting for the kWh change, extra trees for the other three. Honest reading: a month the model
never saw costs about a third more kWh error (3.00 against 2.26), still a factor 11 below the constant; the comfort and
air targets lose more (R² 0.70–0.82), which is why EnergyPlus decides comfort.

Artifact: `data/surrogate_v3/model_v2/v3_surrogate_v2.joblib` (compacted, xz; 69.7 MB, sha256 `fd6db8ee…`, prediction
fingerprint `d9e15808…`, both pinned in `scripts/v3_surrogate.py` and checked before and after unpickling), with
`metadata.json` (features, ranges, both comparisons, both datasets' hashes, the lock hash, the held-out periods).
Out-of-range fallback unchanged (any input more than 10 % of its training span outside the training min–max).

## 6. Evaluation

Sections 6 and 7 describe v2 **without** the owner options (2026-10-18); since the options exist those runs are labelled
`*_v2_plain` (`tmp/v3_allseason/<period>/runs/manual_surrogate_v2_plain.json`, `flexible_auto_v2_plain.json`) and are
reproduced with `--no-v2-options`. The options are in section 8.

### 6.1 Truth grids (EnergyPlus projects all 242 options at each workday decision)

Winter truth: `python -m scripts.v3_surrogate_eval --run-dir tmp/v3_allseason/winter/runs/manual_energyplus/runs/2001-12-14_demo_14d
--out data/surrogate_v3/eval_winter --truth-only --workers 8`, at the state of the manual EnergyPlus run (as for March):
2,430 EnergyPlus runs in 81 min. June and March reuse their truth grids (`data/surrogate_v3/eval`, `eval_spring`). Metrics:
`metrics_v2.json/.md` (and `metrics_v1` for December) beside each truth file.

| | **December v2** | December v1 | June v2 | June v1 | March v2 | March v1 |
|---|---|---|---|---|---|---|
| kWh change MAE (R²) | 1.36 (0.71) | 7.57 (−5.79) | 5.22 (0.976) | 6.68 (0.960) | 2.63 (0.76) | 3.21 (0.67) |
| Peak change MAE kW (R²) | 1.08 (0.91) | 7.53 (−2.06) | 3.10 (0.905) | 4.72 (0.800) | 1.52 (0.93) | 3.23 (0.70) |
| Outside-comfort hours MAE | 0.006 h | 0.60 h | 0.62 h (R² 0.94) | 0.65 h (0.93) | 0.12 h (R² 0.68) | 0.60 h |
| Max occupied air MAE | 0.23 °C (R² 0.63) | 0.27 °C | 0.005 °C | 0.006 °C | 0.027 °C | 0.021 °C |
| Rank correlation (kWh) | 0.85 | 0.80 | 0.995 | 0.993 | 0.91 | 0.89 |
| Live fallbacks (workdays) | **1 / 10** | 10 / 10 | 0 / 10 | 0 / 10 | 1 / 10 | 10 / 10 |
| Top-3 recall | 5 / 10 | 0 / 10 | 10 / 10 | 10 / 10 | 4 / 10 | 1 / 10 |
| Regret of the top 3 (mean / max kWh) | 0.22 / 2.21 | 0.84 / 3.81 | 0 / 0 | 0 / 0 | 0.26 / 1.54 | 1.18 / 3.31 |
| **Regret of the v2 confirmed set (top 3 + 25.5 and 26.0 °C holds)** | **0 / 0** | – | **0 / 0** | – | **0 / 0** | – |

December's R² values are low because the truth spread is tiny (savings of 0–19 kWh a day; the outside-comfort truth is
almost always zero, so its R² is meaningless). The one v2 fallback in December and in March is the last day of the run:
its horizon is cut by the end of the run, so the baseline horizon energy (461–470 kWh) lies below anything in training
(complete horizons only, 1,372 kWh and up); the widened fallback still projected the rule's holds, and the choice was the
same. Speed: 0.10 ms per candidate against 9.8 s per EnergyPlus run (about 100,000×). The rebuild check (`--version v2
--refit`) reproduces the pinned prediction fingerprint exactly.

On the truth grids the rule-inclusive set removes the cost of the surrogate's ranking misses entirely: wherever the top 3
missed the true best, the 25.5 or 26.0 °C full-day hold (always confirmed by v2) saved as much as the true best (in
December the true best sometimes ended at 17:00 or 15:00, with the same kWh within 0.001: little cooling runs in the last
winter hours). Regret of the confirmed set: 0.000 kWh on all 30 decisions.

### 6.2 Fortnights (`scripts/v3_validation_fortnights`, the live demo runtime, verified against no-action replays)

Summaries: `tmp/v3_allseason/<period>/runs/*.json`; table generator `python -m scripts.v3_allseason_report`. The v2 runs
and the June v1 reproductions ran on the final optimizer code (after the v2 pin). The rule, detection-list and winter v1
runs ran earlier the same evening, before the last runtime edits (the v2 envelope preference and the v2 pin), which do not
touch their paths; the June and March rule re-runs are identical to the stored runs. Later edits touched only reporting,
help texts, docs and tests.
Approvals: manual runs approve every HVAC proposal; Flexible AUTO runs grant one envelope permission at step 12 of day 1
and approve nothing by hand; the rule runs record the rule's action as an approved decision (role `rule`).

**14–27 December (primary).** No action 6,702.4 kWh.

| Run | Verified saving (HVAC / lighting) | Used | Below no action | Warmest occupied | Outside 19.5–26 °C | Fallbacks |
|---|---|---|---|---|---|---|
| Manual, detection list (EnergyPlus) | −24.4 (−24.4 / 0) | 6,678.8 | 23.6 | 25.0 °C | 0 h | – |
| Manual, surrogate v1 | −24.4 (−24.4 / 0) | 6,678.8 | 23.6 | 25.0 °C | 0 h | 10 |
| Flexible AUTO, surrogate v1 | −65.1 (−15.7 / −49.3) | 6,629.2 | 73.2 | 25.0 °C | 0 h | 10 |
| **AI manual, v2** | −46.7 (−46.7 / 0) | 6,657.9 | 44.5 | 26.0 °C | 0 h | 1 |
| **Flexible AUTO, v2** | −95.0 (−45.9 / −49.1) | 6,608.5 | 93.9 | 26.0 °C | 0 h | 1 |
| Fixed rule 25.5 °C 09:00–18:00 | −52.4 (−52.4 / 0) | 6,653.0 | **49.4** | 25.5 °C | 0 h | – |
| Fixed rule + lighting delay | −100.8 (−51.2 / −49.5) | 6,603.9 | **98.5** | 25.5 °C | 0 h | – |

| Day | v2 AI manual (verified kWh) | Fixed rule | v2's EnergyPlus projection: 25.5 / 26.0 °C |
|---|---|---|---|
| Sun 16 (max 17.8 °C) | 26 °C 09–18 −2.1 | −2.4 | −4.9 / −4.9 |
| Mon 17 (16.0 °C) | no HVAC action (nothing reaches the 2 kWh / 0.5 % floor) | −0.7 | −0.9 / −0.9 |
| Tue 18 (20.0 °C) | no HVAC action | −2.4 | −0.5 / −0.5 |
| Wed 19 (18.0 °C) | 26 °C **09–17** −1.8 | −1.7 | −2.4 / −2.5 |
| Thu 20 (21.9 °C) | no HVAC action | −3.1 | −1.4 / −1.4 |
| Sun 23 (19.4 °C) | no HVAC action | −0.4 | −0.0 / −0.0 |
| Mon 24 (24.0 °C) | no HVAC action | −3.6 | −0.5 / −0.5 |
| Tue 25 (27.2 °C) | 26 °C 09–18 −8.5 | −7.9 | −4.8 / −5.2 |
| Wed 26 (28.4 °C) | 26 °C 09–18 −14.7 | −13.0 | −11.1 / −12.9 |
| Thu 27 (29.0 °C) | 26 °C 09–18 −19.7 (fallback: cut horizon) | −17.3 | −17.5 / −19.8 |

Why v2 trails here: on the five cold days EnergyPlus projects every option below the benefit floor (≤ 1.4 kWh), so v2
takes no HVAC action by design; the rule acts anyway and its holds verify −0.4 to −3.6 kWh (−10.2 kWh together: real
weather was warmer than the persistence forecast). On the five warmer days v2's 26 °C holds verify 4.5 kWh more than the
rule. Net: 4.9 kWh more energy used than the rule. The Flexible AUTO run behaves the same (its lighting delay ran every
workday, as the rule + lighting run's did).

**8–21 June (validated).** v2 manual and v2 Flexible AUTO chose "hold 25.5 °C 09:00–18:00" on all ten workdays (the
26.0 °C hold is projected to break comfort every day), exactly the rule: −1,565.7 kWh (14,156.9 used) and −1,629.1 kWh
(14,089.2 used), identical to the rule and the rule + lighting runs, verification for verification. Weekends no action.

**2–15 March (seen once before).**

| Run | Verified saving (HVAC / lighting) | Used | Below no action | Warmest occupied | Outside 19.5–26 °C |
|---|---|---|---|---|---|
| v1: manual (EnergyPlus list or surrogate, identical) | −72.5 | 7,049.3 | 66.9 | 25.0 °C | 0 h |
| v1: Flexible AUTO | −118.6 | 7,001.9 | 114.3 | 25.0 °C | 0 h |
| **v2 AI manual** | −138.9 (−138.9 / 0) | 6,991.6 | **124.6** | 26.0 °C | **0.17 h** |
| **v2 Flexible AUTO** | −127.7 (−91.0 / −36.7) | 6,997.2 | 119.0 | 26.0 °C | **0.17 h** |
| Fixed rule | −132.6 | 6,996.5 | 119.7 | 25.5 °C | 0 h |
| Fixed rule + lighting delay | −182.3 (−131.4 / −50.9) | 6,945.8 | **170.4** | 25.5 °C | 0 h |

(v1 rows from `docs/v3_validation_march.md`; the rule rows re-run on the final code, identical.) v2 held 25.5 °C on six
workdays and 26.0 °C on four (Thu 8, Sun 11, Mon 12, Tue 13), where EnergyPlus projected comfort kept. On **Tue 13
March** the real weather was warmer than the forecast and zone L3_South reached **26.000013 °C** for two 5-minute steps
(0.17 occupied zone-hours, 1.3×10⁻⁵ K above the band). In the manual run its verification failed the comfort check; in
the Flexible run the live comfort guard **suspended the permission at once**, as designed (a suspension waits for the
operator and never resumes by itself), so Wed 14 and Thu 15 stayed manual proposals and expired, and the lighting delay
stopped too: −51.4 kWh against the rule + lighting.

### 6.3 Does the AI adapt by season? Yes.

| Period | v2 decisions on the ten workdays |
|---|---|
| June (hot) | 25.5 °C 09:00–18:00 every day (26.0 °C projected to break comfort) |
| March (mild) | 25.5 °C on six days, 26.0 °C on four |
| December (winter) | 26.0 °C on five days (once with an earlier end, 09:00–17:00), **no HVAC action on five cold days** |

The rule does the same 25.5 °C hold every workday in every season.

## 7. Proven and not proven

**Proven (on this model, simulated):**

- The test periods were locked before any v2 data existed (commit `21b6c43`), the lock is enforced in code, and no
  training row touches a test day; the winter test was not touched by any training run.
- The surrogate now covers all twelve months: on the new winter test it falls back on 1 of 10 workdays (the cut-horizon
  last day) instead of 10, with far better accuracy than v1 there (kWh MAE 1.36 against 7.57), and slightly better
  accuracy than v1 on June and March.
- With the rule's holds always confirmed, the v2 confirmed set reaches the true best option on all 30 truth-grid
  decisions (regret 0.0), and in all 50 live v2 decisions where the rule's projection was beneficial the chosen action
  was projected at least as good as the rule (0 exceptions).
- The AI adapts by season (setpoint, window, and no action on cold days).
- The validated June path is unchanged: re-run on the final code, v1 manual −1,565.7 kWh and v1 Flexible AUTO −1,629.1
  kWh match the stored validated runs command for command (status, action, projection, verification); the detection-list
  manual run gives −1,072.9 kWh as before; the fixed-rule runs of June and March are identical to the stored ones.
- Every guard behaved as designed, including the live comfort guard suspending AUTO on a 1.3×10⁻⁵ K overshoot.

**Not proven, or disproved:**

- **"Never worse than the fixed rule" is not achieved in verified energy.** It holds in June (equal) and for the manual
  AI in March; it fails in December (by 4.9 and 4.6 kWh, 0.07 % of the fortnight) and for Flexible AUTO in March (by
  51.4 kWh). Two mechanisms, both measured: (1) the 2 kWh / 0.5 % benefit floor (a pre-execution check of the
  permission) stops the AI on cold days where the rule's small saving is real; (2) the 26.0 °C hold sits exactly on the
  comfort bound, so a forecast error can push a zone 10⁻⁵ K over it, which fails the comfort check and suspends AUTO.
- That a projection-level guarantee carries over to verified results: projections use yesterday's weather.
- Any claim about the December result after this report: the winter test is now seen. A policy change prompted by it
  (below) needs a new untouched period to be tested honestly.
- June and March are no longer out-of-season tests: neighbouring days (1–7 and 22–30 June, 16–31 March, February) are
  now training data, and two training warm-ups pass through their days. March was also seen once before.
- Anything beyond this model: one building, one weather file, simulated occupancy; whether 26 °C is acceptable to
  occupants is a policy choice.

**Owner decisions that would close the gap (not implemented; each widens nothing but changes a documented check):**
(a) let the optimizer pick the 26.0 °C hold only with a comfort margin (for example a projected warmest occupied zone at
most 25.9 °C), which would have kept March at 25.5 °C on Tue 13; (b) let the rule's own 25.5 °C hold through below the
benefit floor when its projection keeps comfort and does not raise the peak, which would have matched the rule on the five
cold December days. Either needs the owner's decision (the benefit floor and the comfort band are part of the
permission's checks) and a new untouched test period.

The validated optimizer (`energyplus`) stays the default of the API, and a double-click of the launcher still starts the
v1 surrogate (`surrogate`); v2 is opt-in (`--optimizer surrogate_v2`, `ENERTWIN_V3_OPTIMIZER=surrogate_v2`) until the
owner decides.

## 8. The owner options (owner decision 2026-10-19) and the February test

The owner chose both options of section 7, behind the v2 flag (`dr.V2_OPTIONS`; run setting `v2_options`, on by
default for `surrogate_v2`, pinned in the run's manifest; `--no-v2-options` reproduces the v2 runs above, which are now
labelled "v2 without options", `*_v2_plain`):

- **(a) Comfort margin:** a 26.0 °C hold (any window, or after a pre-cool) is chosen only when EnergyPlus projects the
  warmest occupied zone at most 25.9 °C (`_edge_without_margin`).
- **(b) The rule below the floor:** when no candidate reaches the 2 kWh / 0.5 % benefit floor, the fixed rule's own 25.5 °C
  hold of that decision is chosen when its projection still saves energy (kWh change < 0), keeps occupied comfort and
  does not raise the day peak (role `rule`); under the Flexible permission AUTO's benefit check accepts exactly this
  action on exactly these conditions (`auto_assessment(..., rule_floor_id=...)`), the audit says so ("projected saving
  below the 2 kWh / 0.5 % floor: the fixed rule's own hold (owner option b, 2026-10-19)"), and the grant states both
  options (`terms.v2_options`, the `requires` text). Every other check and guard is unchanged (comfort band, peak, live
  comfort guard, early read, verification suspension, limits, scope).

Unit tests: `tests/test_v3_optimizer_v2.OwnerOptionTests`.

### 8.1 The February test (locked before the options were coded)

No 14-day window was left without training rows after the all-season v2, so the new test is **held out of a refit**:
`february_fortnight`, Fri 2 – Thu 15 February (one TMYx source year, 2024; seven cold workdays with a daily max of
15.1–21.0 °C, then 26.0 and 29.0 °C on Wed 14 and Thu 15), locked in `config/v3_test_periods.json` (commit `8050a2f`)
before the options existed in code, before any sizing or run of these days and before the hold-out artifact. It binds
`v3_surrogate_v2h`: the v2 recipe (same rows, features, seeds and per-target model choice) **without any row of
`train_february`** (18,208 rows, eleven months; `data/surrogate_v3/model_v2h/`, sha256 `e0dc9490…`, fingerprint
`7cf5080c…`, pinned). The shipped v2 (trained with February) is not used on it. Sizing: HVAC sizes identical, 0 unmet
hours, no-action fortnight **6,862.2 kWh** (cooling 570.1 kWh); stepping matches batch on 4,032 steps (zone air 6.7×10⁻⁶ °C).

| 2–15 February (no action 6,862.2 kWh) | Verified saving (HVAC / lighting) | Used | Below no action | Warmest occupied | Fallbacks (workdays) |
|---|---|---|---|---|---|
| Manual, detection list (EnergyPlus) | −5.1 | 6,857.0 | 5.2 | 25.0 °C | – |
| v2 without options, manual | −6.7 | 6,855.5 | 6.7 | 26.0 °C | 7 |
| v2 without options, Flexible AUTO | −51.4 (−6.6 / −44.8) | 6,810.9 | 51.3 | 26.0 °C | 7 |
| **v2 + options, manual** | −10.8 | 6,850.9 | 11.3 | 25.5 °C | 7 |
| **v2 + options, Flexible AUTO** | −55.6 (−11.0 / −44.6) | 6,806.2 | 56.0 | 25.5 °C | 7 |
| Fixed rule | −11.8 | 6,849.9 | **12.3** | 25.5 °C | – |
| Fixed rule + lighting delay | −56.5 (−12.0 / −44.6) | 6,805.3 | **56.9** | 25.5 °C | – |

0 occupied zone-hours outside 19.5–26 °C in every run.

- **With the options the AI trails the rule by 1.0 kWh (manual) and 0.9 kWh (Flexible AUTO); without them by 5.6 kWh.**
  The options ran the rule's hold on eight of ten workdays: seven below the floor (projected savings 0.05–0.85 kWh) and
  Thu 15 (−3.9 kWh projected, above the floor). The remaining gap is Sun 11 and Tue 13 February: the rule's hold was projected at exactly
  0.0 kWh, not a saving, so option (b) did not apply; the rule acted anyway and verified −0.6 and −0.4 kWh.
- The hold-out surrogate fell back on 7 of 10 workdays: without February, the coldest building states (morning zone air
  20.0–21.4 °C) lie below its training range. The widened fallback still projected the rule's holds, so the choice was
  unaffected. Truth grid (2,430 EnergyPlus runs, `data/surrogate_v3/eval_february/metrics_v2h.md`): savings are tiny
  (true best 0.72 kWh a day on average), so accuracy figures mean little (kWh MAE 1.04 against a truth spread of 1.00);
  regret of the v2 confirmed set 0.0; the choice replayed on the truth: v2 without options −4.06 kWh, **with options
  −7.03 kWh = the rule's −7.03**, true best −7.19 kWh.

### 8.2 The options on the earlier periods (seen; shipped v2)

| Period | v2 + options, manual | Fixed rule | v2 + options, Flexible AUTO | Fixed rule + lighting |
|---|---|---|---|---|
| 14–27 Dec | 6,653.0 (**= rule**) | 6,653.0 | 6,665.3 (**trails by 61.4**) | 6,603.9 |
| 8–21 Jun | 14,156.9 (= rule) | 14,156.9 | 14,089.2 (= rule + lighting) | 14,089.2 |
| 2–15 Mar | 6,996.5 (**= rule**) | 6,996.5 | 6,945.8 (**= rule + lighting**) | 6,945.8 |

(kWh used.) Option (a) removed the March overshoot (warmest occupied 25.5 °C, 0 h outside, no suspension): March now
matches the rule in both modes. Option (b) closed the December manual gap. **But December Flexible AUTO lost 61.4 kWh:**
on Sun 23 December the rule's hold ran under option (b) with a projected saving of 0.002 kWh; the day was warmer than the
persistence forecast, and after 455 minutes the realized energy was 5.1 kWh above the no-action *forecast* (early-read
limit 5.0 kWh), so the early-read guard suspended the permission (it verified −0.2 kWh: a real saving against the
no-action replay). AUTO then waited for an operator and Mon 24 – Thu 27 expired. A hold with almost no projected saving
has no buffer against forecast error, and the early read compares against the forecast.

### 8.3 Verdict with the options

| | Manual AI against the rule | Flexible AUTO against rule + lighting |
|---|---|---|
| **February (untouched)** | trails by 1.0 kWh | trails by 0.9 kWh |
| December | equal | trails by 61.4 kWh (early-read suspension) |
| June | equal | equal |
| March | equal | equal |

- With the options the manual AI is **never worse than the rule by more than 1.0 kWh** (0.015 % of the fortnight) in
  four periods and equal in three; the Flexible AUTO equals the rule + lighting in June and March, trails by 0.9 kWh in
  February and by 61.4 kWh in December. The milestone "never worse" is therefore still **not met exactly**, and the AI
  with the options is in practice the fixed rule: in the 40 workdays of the manual runs it chose something other than the
  rule's hold three times (a 26.0 °C hold 09:00–17:00 with margin on Wed 19 December; no action on Sun 11 and Tue 13
  February, where the rule's projection was exactly zero).
- Two mechanisms remain, both measured: (1) a rule hold projected at exactly 0.0 kWh is not "a saving" under option (b),
  yet the rule's verified saving that day was small and real; (2) under AUTO, a below-floor rule hold can trip the early
  read on a day warmer than forecast, and the suspension then costs every remaining day.
- Owner decisions that would close them (not implemented, each changes a guard or a check): accept the rule's hold when
  its projection is ≤ 0 kWh (not only < 0); and for a below-floor rule hold, read early against the no-action replay
  instead of the forecast, or exempt it from the early-read loss limit while keeping the live comfort guard. Either
  needs another untouched period to test honestly, which this weather year no longer has without another hold-out.
