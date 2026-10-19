# v3 ML surrogate of the EnergyPlus decision step, and the envelope AUTO permission (2026-10-17)

The v3 optimizer projects every candidate action with an isolated EnergyPlus run, so a decision could afford only four
to six candidates. This milestone adds a trained ML surrogate of that forecast-time projection: it screens a documented
grid of 242 candidate actions in milliseconds, EnergyPlus confirms the few it picks with the existing projection, and
every downstream check (comfort, thresholds, approval or AUTO, verification) reads EnergyPlus figures only. The second
part is the envelope AUTO permission: within the documented candidate space the system runs 14 simulated days on one
grant with no daily approvals. Simulated control of a simulated building; nothing physical is controlled.

**The surrogate never approves, executes or verifies anything.** It ranks. EnergyPlus is the final check: a candidate the
surrogate screened out has no projection and can never be proposed or executed; a candidate it ranked first is proposed
or executed only after EnergyPlus projects it beneficial and comfortable.

**Single source of truth for the numbers:** `docs/v3_results.md` (the four fortnights, the share of the no-action
fortnight, the limits).

**Results in one paragraph.** Trained on 11,664 EnergyPlus projections from six months other than June, the surrogate
put EnergyPlus's true best action first on all ten held-out workdays of 8–21 June (top-3 recall 10/10, regret 0 kWh) at
about 0.1 ms per candidate against about 11 s per EnergyPlus run. On one envelope grant with zero approvals the
fortnight saved **−1,629.1 kWh**, against −1,138.8 kWh for the exact-hold AUTO run and −1,072.9 kWh manual (both
reproduced exactly on the final code): +43 %, by holding 25.5 °C instead of 25.0 °C, inside the comfort band. That is
10.4 % of the 15,678.3 kWh the same fortnight uses with no action. The same 25.5 °C hold won every workday, so a fixed
rule would have matched it on this fortnight.

## 1. Decision flow

Current optimizer (`optimizer: energyplus`, the default, unchanged): no-action EnergyPlus baseline → the detection rules'
4–6 candidates → one EnergyPlus run each → recommendation → proposal, AUTO guard or no action.

Surrogate optimizer (`optimizer: surrogate`, `--optimizer surrogate` on the runtime and the launcher,
`ENERTWIN_V3_OPTIMIZER` for the API):

1. the no-action EnergyPlus baseline, unchanged (it is the forecast, and its figures are inputs of the surrogate);
2. the candidate grid of section 2 replaces the detection list on a workday (the detection texts stay);
3. the surrogate estimates per candidate the kWh change over the horizon, the issue-day peak change, the new occupied
   zone-hours outside 19.5–26 °C and the maximum occupied air temperature (`scripts/v3_surrogate.screen`);
4. candidates whose estimate shows new time outside the band (> 0.1 h) or a raised peak (> 0.5 kW) are screened out;
   the rest are ranked by estimated saving; the **top 3** cooling candidates, the exact-hold action
   (`occupied_setpoint_25`, only while the exact-hold permission holds, so it can act) and the best lighting estimate are
   confirmed by EnergyPlus exactly as today;
5. from here nothing changes: `projected` comes only from EnergyPlus; thresholds, `beneficial`, `in_scope`, the
   pre-execution guard, proposals, approvals, AUTO execution and verification read EnergyPlus figures only.

Fallback: a run with `optimizer: surrogate` refuses to start without the artifact. At a decision, any feature outside the
training range (more than 10 % of its training span beyond the min–max), an issue time outside the trained domain
(09:00–13:00; the live AI check is 09:00), a non-workday or a model error records `screening.status: fallback` with the
reason and evaluates the detection list exactly as the current optimizer does.

Record: every decision carries a compact `screening` block (status, screened count, confirmed ids, the estimates of the
confirmed candidates, the eight best estimates, the artifact version and sha256, milliseconds per candidate, the chosen
ids). Confirmed candidates carry both `estimate` and `projected`; screened-out ones listed for the operator carry
`estimate` and `skipped: "screened out by the ML surrogate (estimate only, not confirmed by EnergyPlus)"`.

## 2. Candidate space (documented bounds, `scripts/v3_surrogate.grid`)

All on the issue day; every window starts at or after the issue step and HVAC on (07:00) and ends at or before HVAC
off (18:00). Nothing crosses midnight. Values stay inside the thermostat range enforced at approval time (15–32 °C).

| Family | Parameters | Bounds | Why |
|---|---|---|---|
| Cooling hold | setpoint, start, end | setpoint ∈ {24.5, 25.0, 25.5, 26.0} °C (baseline 24.0); start ∈ {issue step, 10:00, 11:00, 12:00, 13:00, 14:00, 15:00}; end ∈ {15:00, 16:00, 17:00, 18:00}, end ≥ start + 1 h | 26.0 °C is the top of the comfort band: EnergyPlus rejects it (zone air overshoots the setpoint by about 0.0004 °C), so the grid stops there and the surrogate learns the boundary. An end before 18:00 shows the snap-back rebound the current optimizer already evaluates |
| Pre-cool, then hold | pre setpoint, pre duration, hold setpoint | pre ∈ {22.5, 23.0, 23.5} °C for 1 h or 2 h, immediately before a hold at {24.5, 25.0, 25.5, 26.0} °C ending 18:00; the pre-window starts at or after the issue step; the hold lasts at least 3 h | The current `precool_then_hold_25` generalised |
| Lighting | absence delay | {5, 10} min (baseline 15) | `_safe` allows 1–3 steps; 15 is the baseline |
| No action | — | the reference every estimate is relative to | |

At a 09:00 issue: 108 holds, 132 pre-cool variants and 2 lighting actions, 242 candidates (tested). Ids are deterministic
(`hold_25.5_1000_1600`, `precool_23.0_1h_hold_25.5_1100_1800`, `lighting_delay_10`); a grid action identical to an
existing one keeps the existing id and title (`occupied_setpoint_25`, `lighting_delay_5`), so the exact-hold scope rule
reads it as before. The same bounds are the envelope permission's rule (`sg.ENVELOPE`, `sg.in_envelope`).

## 3. What the surrogate learns

It emulates `summarize(simulate(action)) − summarize(simulate(None))` of the existing projection, from 63 inputs that
exist at the issue step (`scripts/v3_surrogate.features`; a test passes truncated records and asserts identical features,
so nothing at or after the issue step is used):

- **thermal state** (persisted steps before the issue step): zone air of the 15 zones at the last step; min, mean and
  max over occupied zones; HVAC kW at the last step and kWh over the last hour; the day's kWh so far; the cooling
  setpoint in effect; yesterday's override hours and mean setpoint delta (the rebound carried in);
- **forecast weather assumptions** (the persistence forecast: the previous day's hours): the previous day's dry bulb at
  10, 12, 14, 16, 18 and 21 h, its maximum and its global horizontal radiation sum; the issue day's observed dry bulb at
  hours 7 and 8 and at the issue hour;
- **calendar:** day of year (sine, cosine), weekday, issue step;
- **expected occupancy** (the occupancy model's previous same-type days, as the forecast): expected person-hours for the
  rest of the day, mean head count 09:00–18:00, expected lit-while-empty room-minutes;
- **baseline forecast** (the no-action EnergyPlus run of the same decision, which runs anyway): rest-of-issue-day kWh,
  HVAC kWh 09:00–18:00, day peak kW and its step, maximum occupied air, horizon kWh, HVAC kWh inside the candidate's
  window and in the hour after it;
- **action:** family one-hot, setpoint and its delta from 24 °C, start, end, hold hours, pre setpoint, pre hours,
  lighting delay, "ends at HVAC off".

Targets: `kwh_change` (whole horizon: the action's day and three more, then on to the end of the next workday),
`peak_change_kw` (issue day), `outside_hours_delta` (new occupied zone-hours outside 19.5–26.0 °C) and
`occupied_air_max_c`.

## 4. Training data (held-out fortnight untouched)

**Periods** (`v3.PERIODS`, not demo periods: `period_for` ignores them, so the demo and the shadow cannot run them):
`train_april` (1–30 Apr), `train_may` (6–31 May), `train_july` (8–31 Jul), `train_august` (5–31 Aug), `train_september`
(2–30 Sep), `train_october` (7–31 Oct). Each lies inside one TMYx source month (Apr 2013, May 2011, Jul 2015, Aug 2015,
Sep 2015, Oct 2014: no weather seam inside a period) and its 14-day lead-in and its horizons stay clear of 8–21 June
(tested). June is excluded entirely.

**Reference sizing** per period with the validated method (`python -m scripts.energy_model_v3_occupancy --period
train_*`: reference run, occ_zone and occ_room runs, the acceptance checks). The building and the physics are untouched;
sizing comes from the design days:

| Period | Days | HVAC sizes identical to reference | Max zone occupant-hours error | Lights kWh error | Unmet occupied cooling hours (occ_room, whole run period) | Max occupied zone air (reported days) |
|---|---|---|---|---|---|---|
| train_april | 30 | yes | 0.0065 | 0.0 | 0.00 | 24.00 °C |
| train_may | 26 | yes | 0.0060 | 0.0 | 0.00 | 24.00 °C |
| train_july | 24 | yes | 0.0054 | 0.0 | 0.00 | 24.13 °C |
| train_august | 27 | yes | 0.0058 | 0.0 | 5.08 | 24.77 °C |
| train_september | 29 | yes | 0.0064 | 0.0 | 3.17 | 24.05 °C |
| train_october | 25 | yes | 0.0058 | 0.0 | 0.00 | 24.00 °C |

**Context runs** (`scripts/v3_surrogate_data.context_run`): per period two persisted runs in the demo step format, so
`cf.simulate` replays them exactly (every row's `pre_issue_check.max_abs_diff` is 0.0): (a) no action on any day; (b)
on each workday, with a seeded random choice, one random cooling action of the grid (probability 0.7) and one random
vacancy-delay action (0.5), so morning states carry yesterday's rebound as live runs do.

**Samples:** at the 09:00 issue step of every workday whose horizon is complete inside the period, a seeded random subset
of 40 candidates (always including `occupied_setpoint_25`, `lighting_delay_5` and `lighting_delay_10`) plus the
baseline, in both contexts; one anchor day per period gets the whole grid; a third of the days also get an 11:00 or
13:00 issue with 20 candidates (in the random context only on days without a context action, so a sample never sits
inside a running action it does not know about). All EnergyPlus, through the optimizer's own `cf.simulate`, in a
process pool with fixed seeds (`SEED = 20261017`). The generator is one reproducible, resumable script
(`python -m scripts.v3_surrogate_data`) with its own manifest (`data/surrogate_v3/dataset_manifest.json`: periods, seeds,
counts, code hashes, context-run manifests, dataset sha256).

**Pilot and sizing:** one decision of train_april (49 runs) took 46 s on 24 workers after a 38 s context run; a
late-May probe (longer replay) took 60 s for 49 runs (1.22 s per run of throughput). The full plan was sized to 252
decisions and 11,916 EnergyPlus runs.

**Running it on this laptop (32 threads, 31.2 GB RAM, 12 GB page file):** the binding limit is the Windows commit limit
(43.2 GB), not physical RAM. The first attempt at 30 workers ended in an EnergyPlus `Severe ** bad allocation` while
other programs held about 17 GB. Two changes made it safe: the generator's pools recycle workers every four runs
(`max_tasks_per_child=4`), and no worker imports numpy (importing numpy commits about 1.5 GB per process for OpenBLAS
buffers with 32 threads: 1,584 MB against 106 MB with `OPENBLAS_NUM_THREADS=1`). The run was resumed several times
(stops, a restart of the laptop); the resume skips finished decisions, and a check after every stop found the rows and
the progress file consistent (no torn line, no orphan rows, no duplicates). 14 to 21 workers ran at about 36–48
EnergyPlus runs per minute.

**Dataset** (`data/surrogate_v3/dataset.jsonl`, sha256 `59594cd3…`): 252 decisions, **11,664 rows** over 210
period-context-days; every row's replay of the persisted past was exact (`pre_issue_check.max_abs_diff` 0.0).

| | Rows |
|---|---|
| train_april none / random | 1,142 / 1,022 |
| train_may none / random | 1,002 / 902 |
| train_july none / random | 902 / 822 |
| train_august none / random | 1,002 / 882 |
| train_september none / random | 1,142 / 1,042 |
| train_october none / random | 962 / 842 |
| Issue 09:00 / 11:00 / 13:00 | 10,824 / 440 / 400 |
| Holds / pre-cools / lighting | 5,183 / 5,977 / 504 |

| Target | Min | Median | Max | Mean |
|---|---|---|---|---|
| kWh change over the horizon | −244.9 | −34.3 | +32.4 | −46.4 |
| Issue-day peak change (kW) | −35.2 | +6.2 | +45.7 | +6.8 |
| New occupied zone-hours outside 19.5–26 °C | 0.0 | 0.0 | 40.7 | 1.8 |
| Max occupied air (°C) | 24.0 | 25.0 | 26.0 | 25.2 |

22.9 % of the candidates leave the comfort band somewhere (mostly the 26.0 °C holds and short holds that snap back) and
66.5 % raise the issue-day peak (mostly holds that end before 18:00): the surrogate has to learn both boundaries.

## 5. Model choice

Compared with grouped cross-validation (5 folds, groups = calendar day, so a day's candidates never sit on both sides of
a fold), one model per target (`scripts/v3_surrogate_train.py`, scikit-learn 1.9.1, fixed seeds): a constant
predictor, ridge regression, k-nearest neighbours, histogram gradient boosting, extra trees and a small MLP. The pick
per target is the lowest out-of-fold MAE among models that predict in under 1 ms per candidate.

Out-of-fold results (5 folds grouped by day, 11,664 rows; prediction time measured while the comparison ran 16 cells in
parallel, so it is an upper bound):

| Target | Constant | Ridge | kNN | Gradient boosting (HGBR) | Extra trees | MLP |
|---|---|---|---|---|---|---|
| kWh change: MAE (R²) | 35.80 (−0.01) | 13.54 (0.677) | 7.99 (0.926) | **1.71 (0.997)** | 2.02 (0.995) | 6.93 (−5.78) |
| Peak change kW: MAE (R²) | 10.39 (−0.01) | 5.72 (0.683) | 3.41 (0.852) | 1.58 (0.970) | **1.24 (0.973)** | 3.64 (−13.2) |
| Outside hours: MAE (R²) | 2.79 (0.00) | 2.46 (0.375) | 1.13 (0.657) | **0.314 (0.942)** | 0.319 (0.941) | 0.60 (0.434) |
| Max occupied air °C: MAE (R²) | 0.516 (0.00) | 0.017 (0.997) | 0.124 (0.909) | 0.0027 (0.9996) | **0.0022 (0.9997)** | 1.79 (−4,095) |
| Prediction ms per row | 0.0001 | 0.0005 | 0.05 | 0.02 | 0.05 | 0.0015 |

Choice (lowest MAE under 1 ms per candidate): gradient boosting for the kWh change and the outside hours, extra trees
for the peak change and the maximum occupied air. Honest reading: ridge already beats the constant by a factor of
2.6 on kWh, so the inputs carry the signal; the tree ensembles cut the error by another factor of 6–8 because the
effects are non-linear (comfort boundary at 26 °C, snap-back peaks of short holds, rebound after pre-cools). The MLP
diverged on some folds (R² far below zero: a few extreme predictions) and is not used. kNN is the fastest to fit but
4–5 times less accurate than the trees.

The artifact (`v3_surrogate_v1.joblib`, trained at 120 MB, sha256 `db01acee…`; since 2026-10-18 compacted to 39.6 MB,
sha256 `5d40ce76…`, loads in 3.4 s, bit-identical predictions, see `docs/v3_results.md`) took 1.5 minutes to compare and
fit on 16 processes; BLAS runs single-threaded during training and inference (deterministic sums, no 1.5 GB OpenBLAS
commit per process; a test checks that loading the artifact keeps it so).

Artifact: `data/surrogate_v3/model/v3_surrogate_v1.joblib` plus `metadata.json` (features, training ranges, targets,
the comparison, dataset sha256 and row count, code hashes, scikit-learn version, artifact sha256). Inference is
deterministic (tree ensembles run single-threaded; a test reloads the artifact twice and compares predictions exactly,
and refits with the same seed to the same predictions). A surrogate run pins the artifact's version, file, sha256 and
model choice under `settings.surrogate` in its manifest, so a changed artifact cannot resume a run.

## 6. Held-out evaluation (8–21 June)

### 6.1 Accuracy, ranking and speed (`python -m scripts.v3_surrogate_eval`)

Truth: the persisted live fortnight run `2001-06-08_demo_14d` (manifest `b771ff22…`, copied; the live folder is never
written) at 09:00 on each of its ten workdays, the whole grid projected by EnergyPlus: 2,420 candidates plus the ten
baselines (`data/surrogate_v3/eval/truth.jsonl`, sha256 `59bd39de…`). None of these days, nor any day of 8–21 June, is
in the training data. The last three days' horizons are cut by the end of the run, exactly as the live optimizer sees
them.

| Target | MAE | R² | Truth mean | Truth std |
|---|---|---|---|---|
| kWh change over the horizon | 6.68 kWh | 0.960 | −63.1 | 42.8 |
| Issue-day peak change | 4.72 kW | 0.800 | +5.8 | 12.5 |
| New occupied zone-hours outside the band | 0.65 h | 0.934 | 3.2 | 6.7 |
| Max occupied air | 0.0064 °C | 0.999 | 25.24 | 0.56 |

| Ranking (per workday decision, 10 decisions) | Result |
|---|---|
| Top-3 recall: EnergyPlus's true best (comfort kept, peak not raised) among the surrogate's three picks | **10 / 10** |
| Regret of the confirmed set (true kWh of its best member minus the true best) | **0.0 kWh** on every day |
| Spearman rank correlation of estimated and true kWh, cooling candidates | 0.993 (mean) |
| Best lighting delay picked correctly | 10 / 10 |
| True best on every day | `hold_25.5_0900_1800` (hold 25.5 °C 09:00–18:00), mean −152.4 kWh over its horizon |

| Speed | |
|---|---|
| Surrogate | 0.109 ms per candidate (2,420 in 0.26 s, one process); a 242-candidate screen about 26 ms plus feature building |
| EnergyPlus | 11.19 s for one projection run, one process (measured while the two fortnights below ran; the pilot measured 8.1 s on an idle machine) |
| Ratio | about 100,000× per candidate. Projecting the whole grid with EnergyPlus would take about 45 minutes in one process, or about 3 minutes on 24 workers, per decision |

Honest limits: held-out accuracy is lower than in cross-validation (kWh MAE 6.7 against 1.7; peak R² 0.80 against
0.97): June 2020 weather lies inside the training ranges but differs from the six training months, and the held-out
morning states carry the live run's own actions. The estimates also let through more candidates than EnergyPlus accepts
(on 9 of 10 days the surrogate judged 62–83 cooling candidates comfortable against 39–72 by EnergyPlus); that is why
EnergyPlus confirms before anything is proposed or executed. Ranking, the part the surrogate is used for, held up: the
true best was the surrogate's first pick on all ten days.

### 6.2 End to end: 14 days on the held-out fortnight (`ENERTWIN_SURROGATE_TEST=1`)

`tests.test_v3_demo_multiday.SurrogateFortnightTests`: the surrogate optimizer runs the live demo runtime through 8–21
June, decision at 09:00 every day, interrupted on the last step of Sunday 17 June and resumed (2,880 steps replayed,
max diff 0.0, in both runs). Every verification is the usual independent no-action replay over the action's horizon.

| 14 days, 8–21 June | Approvals | Verified saving | Used | Δ against exact-hold AUTO |
|---|---|---|---|---|
| **Envelope AUTO, one grant on day 1, surrogate optimizer** | **0** (0 fallbacks, 0 proposals left) | **−1,629.1 kWh** (holds −1,561.6, lighting −67.5) | 14,089.2 kWh | **−490.3 kWh (+43 %)** |
| Surrogate optimizer, manual (every proposal approved, HVAC only) | 10 | −1,565.7 kWh | 14,156.9 kWh | |
| Exact-hold AUTO, current optimizer (regression rerun, final code) | 0 | −1,138.8 kWh (holds −1,069.2, lighting −69.6) | 14,556.2 kWh | reference |
| Manual, current optimizer (regression rerun, final code) | 10 | −1,072.9 kWh | 14,627.1 kWh | |

- **What the decisions were.** On every one of the ten workdays EnergyPlus confirmed the surrogate's first pick, "hold
  25.5 °C 09:00–18:00", and the lighting pick, "lights off 5 min after a room empties". The weekends were no action,
  with the reason. Verified per hold: −146.4 to −173.0 kWh (projected −145.2 to −165.8); the ten projections sum to
  −1,543.6 kWh against −1,561.6 verified.
- **Better, plainly:** the decisions improved: 43 % more verified saving than the exact-hold AUTO run, with zero
  approvals and every guard in place. The price is comfort margin, inside the owner's band: during a hold the warmest
  occupied zone reaches 25.5 °C instead of 25.0 °C (0 occupied zone-hours outside 19.5–26 °C; every verification's
  comfort and occupancy checks passed). The 25.5 °C hold also cuts the day peak by about 21 kW instead of 15 kW.
- **What the surrogate did not do:** on this fortnight the best choice was the same every day, so a fixed "25.5 °C"
  rule would have done as well here. The surrogate's contribution is that it found that choice among 242 documented
  options at every decision, without a hand-written rule, and would find a different one where the weather, the
  morning state or the rebound make another window or a pre-cool better (the training data contain such days).
- **Last days:** as before, the horizons of the Tuesday to Thursday actions of week 2 are cut by the end of the run
  (`horizon_complete: false`); their later rebound is not simulated, for every run in the table alike.

### 6.3 Second test fortnight, 2–15 March (outside the training range, 2026-10-18)

A pure test, nothing retrained or tuned (`docs/v3_validation_march.md`). On all ten March workdays the inputs fall
outside the training range (morning ground-floor zone air 20.0–20.7 °C; on seven days a 07:00 outdoor temperature of
8.5–14.6 °C), so the live optimizer fell back to the detection list every time. The whole-grid truth (2,430 EnergyPlus
runs, `data/surrogate_v3/eval_spring/metrics.md`) shows why the fallback is needed: top-3 recall 1/10, regret mean
1.18 kWh (max 3.31), kWh MAE 3.21 with R² 0.67, rank correlation 0.894. The true best in March was a 09:00–18:00 hold at
25.5 °C (five days) or 26.0 °C (five days). A fixed "25.5 °C, 09:00–18:00" rule beat both the manual and the
Flexible-permission AUTO runs in March and matched the surrogate runs exactly in June.

## 7. The envelope AUTO permission (owner decision, 2026-10-17)

A third consent scope, `envelope` (`term: multi_day`: 1–14 simulated days, no wall-clock cap, bound to the run and its
manifest, one at a time, never silently renewed; the same per-day limits and guards as the exact-hold permission). Within
it AUTO executes whatever the optimizer selects each day, with no daily approvals:

| | Envelope permission (`scope: envelope`) | Exact-hold permission (`scope: multi_day`, unchanged) |
|---|---|---|
| Cooling | any setpoint level, hold window or pre-cool option of the candidate space (section 2): `COOLINGSETPOINT` only, 24.5–26.0 °C in the hold, 22.5–23.5 °C in a pre-cool of at most 2 h followed by a hold of at least 3 h to 18:00, a window on the issue day inside 07:00–18:00 of a workday | exactly the 25 °C hold 09:00–18:00 on workdays |
| Lighting | any vacancy delay of the candidate space (5 or 10 min) | 5 min only |
| Excluded (refused by `in_scope` → `sg.in_envelope`) | `HVACHOURS` (HVAC off during occupied hours), `HEATINGSETPOINT`, plug loads, anything outside cooling setpoints/schedules and lighting delays; non-workdays; windows across midnight | everything else |
| Confirmation | each executed action is confirmed by EnergyPlus (never by the surrogate) and passes the pre-execution checks: optimizer selected it, projected saving ≥ 2 kWh and 0.5 % (HVAC) or ≥ 0.25 kWh (lighting), projected occupied air within 19.5–26 °C, day peak not raised, safe ranges, one action per channel per day, workday | the same |
| Guards | live comfort guard (both bounds), early read, verification suspension; a suspension waits for the operator and never auto-resumes; revoke at the next step; expiry; audit (`authorization.method: envelope_operator_permission`, consent id and digest, issuer, checks passed, projection); exactly-once; exact replay | the same |

The grant sheet offers the envelope first (on screen "Flexible permission": families, bounds, exclusions, the EnergyPlus
confirmation, the guards), then the exact-hold permission and the short lighting consent. The consent record carries
the envelope bounds (`envelope`), the API's `auto_consent.policy.envelope` and `scopes.envelope` state them, and the
launcher's AUTO line names an active envelope. The statement is in `CLAUDE.md`, `AGENTS.md` and `PROJECT_HANDOVER.md`.
The envelope grant is accepted with either optimizer: the rule is on the action's parameters.

Under the exact-hold permission with the surrogate optimizer, the daily lighting candidate is `lighting_delay_5` only and
the exact hold is always among the confirmed candidates; a better out-of-scope selection is a manual proposal exactly as
today.

## 8. Interface (Dispatch Curve kept)

In the compact "Optimization available" card and at the top of the review sheet (2026-10-18 wording): "242 options
screened by ML surrogate → top N confirmed by EnergyPlus" (N counts the cooling candidates; the lighting pick is
confirmed too). The same line follows an AUTO hold in the AI caption and on its effect card, and the Evaluating caption
says "The ML surrogate screens the candidate grid; EnergyPlus confirms its top picks". In the review sheet under "How
the AI got here": "242 options screened by ML surrogate → top N confirmed by EnergyPlus →
chosen: …", then the confirmed candidates with two labelled figures, **ML estimate** and **EnergyPlus projection**, the
best screened-out estimates marked "not confirmed", and the artifact version. The projected impact above it stays an
EnergyPlus projection; the verified sheet keeps "verified". The existing texts stay for the current optimizer.

**Live browser evidence** (Edge through Playwright CLI, Demo speed, a 14-day run started with
`python -m scripts.start_v3_demo --optimizer surrogate --start --days 14 --speed demo`, signed in as a throwaway test
operator in an auth store of its own under `tmp/v3_surrogate/auth/`; screenshots in `tmp/v3_surrogate/evidence/`):

1. Friday, AUTO off: the grant sheet offers "Flexible permission" (the envelope) first, with its families, bounds, the EnergyPlus
   confirmation, the exclusions and the guards; "Grant for …" stays disabled until the acknowledgement is ticked
   (`e00` desktop, `e01` phone).
2. Sunday 09:00, no permission yet: "Optimization available … 242 options screened by the ML surrogate · 4 confirmed by
   EnergyPlus" (`e02`); the review sheet: "242 options screened by the ML surrogate → 4 confirmed by EnergyPlus →
   chosen: Hold cooling at 25.5 °C, 09:00–18:00 (30 s). The surrogate only ranks; every figure above is an EnergyPlus
   projection", then "Confirmed by EnergyPlus — ML estimate · EnergyPlus projection": the chosen hold est. −158.4 ·
   −154.4 kWh, the 10:00 hold est. −150.2 · −140.5, the pre-cool est. −142.3 · −133.9, the lighting delay est. −6.0 ·
   −6.2, then eight 26 °C options "not confirmed", and "v3_surrogate_v1 · 2.9 ms per option" (`e03`, `e03b` desktop,
   `e04` phone). Approved by the operator (Approve Once).
3. Sunday 18:50, the envelope granted for 14 days: "AUTO flexible permission: day 1 of 14", the envelope stated in the sheet,
   "Ends Thu 21 Jun 24:00", pill "AUTO day 1/14" (`e05`).
4. Monday 09:00: the AI acted on its own under the envelope: the 25.5 °C hold and the 5-minute lighting delay,
   `authorization.method: envelope_operator_permission`; the curve reads "AI check 09:00, AUTO hold" and "AUTO lights
   09:00", the effect card "AUTO under evidence_operator's consent", the caption "Hold cooling at 25.5 °C, 09:00–18:00
   (AUTO)" (`e06`); the active permission at phone width (`e07`).
5. End of the run (29.5 min wall time, 4,032 steps, run `2001-06-08_demo_14d`, optimizer `surrogate` pinned in its
   manifest): 19 actions, **1 manual approval** (Sunday, before the grant) and **18 AUTO** under the one envelope grant
   (nine 25.5 °C holds, nine 5-minute lighting delays), 0 proposals left, the permission `auto_expired` at the end of
   the run. Every action verified with comfort and occupancy kept: **−1,623.6 kWh** of 14,095.6 kWh used; week 1
   −818.8, week 2 −804.8; the last three days flagged "rebound not counted" as before (`e08`).
6. After the run, the grant sheet at 390 px: the three AUTO kinds wrap inside the sheet (`e09`; the first capture `e01`
   showed the third button clipped, fixed in `twin.css`).

The throwaway operator of this evidence run lives in `tmp/v3_surrogate/auth/` (its own auth database and config, a
generated password in a local file, never printed); the normal operator accounts were not used or read.

## 8b. All-season v2 (2026-10-18)

Everything above describes the validated v1 (`--optimizer surrogate`, artifact `v3_surrogate_v1`, pins unchanged). The
all-season retrain is `--optimizer surrogate_v2` with its own artifact (`data/surrogate_v3/model_v2/v3_surrogate_v2.joblib`,
sha256 `fd6db8ee…`, fingerprint `d9e15808…`): the same features, grid and method, trained on all twelve months with the
locked test fortnights excluded (20,092 rows), and an optimizer that always has EnergyPlus confirm the fixed rule's
25.5 °C hold, the 26.0 °C hold and the 5-minute lighting delay, then picks the largest projected saving that is
beneficial and does not raise the peak. Owner options (2026-10-19, on by default for `surrogate_v2`): a 26.0 °C hold
only with a projected warmest occupied zone ≤ 25.9 °C, and the rule's 25.5 °C hold also below the benefit floor when it
still saves, keeps comfort and the peak. A February hold-out refit (`v3_surrogate_v2h`) serves the untouched February
test only. Results, method and limits: `docs/v3_allseason.md`; numbers: `docs/v3_results.md`.

## 9. Limits

- The surrogate is trained on 09:00, 11:00 and 13:00 issues of six training months and on two contexts (no action,
  one random action per day); a decision taken while another action is still running (the live `background`) is not
  described by the features, so its estimate is less accurate. EnergyPlus confirms anyway.
- Estimates are not projections: a screened-out candidate was never simulated, so a surrogate miss costs saving, not
  comfort (comfort is EnergyPlus-checked on every executed action).
- Training data come from the same model and the same persistence weather assumptions as the live runs; a different
  building, weather file or occupancy model needs a new dataset and artifact (`VERSION` bump).
- The 26.0 °C holds are in the grid to teach the comfort boundary; in June EnergyPlus rejects them, as expected (in
  March, a milder fortnight outside the training range, they pass comfort and are the true best on five days).
- A surrogate live run keeps the live run id; switching the optimizer of an existing run is refused by the manifest check
  (Restart archives the old run, never deletes it).
- Held-out absolute accuracy (kWh MAE 6.7, peak R² 0.80) is below the cross-validated figures; the ranking held, but a
  month unlike the six training months could rank less well. EnergyPlus confirmation bounds the harm to lost saving.
  Measured on 2–15 March (6.3): outside the training range it ranks poorly (top-3 recall 1/10); the range check sent
  every decision to the detection list, and no tested fortnight shows the AI beating a fixed 25.5 °C rule.
- Deferred minor: under the surrogate optimizer the runtime still starts the exact-hold run with the baseline even when
  the screen will not confirm it (one wasted EnergyPlus run per decision).
- Resolved 2026-10-18 (final polish, `docs/v3_results.md`): if every cooling estimate fails the screen, the decision now
  falls back to the detection list (it used to offer no cooling candidate); a lighting candidate cut to a short
  consent no longer sends the whole decision to fallback; the artifact is 39.6 MB (was 119.6 MB), pinned by sha256 and
  checked before it is unpickled, with bit-identical predictions; cross-validation now groups by calendar day (more
  conservative figures, the validated artifact kept). All four fortnights reproduce exactly on that code.
- An envelope grant records `duration_days` as asked (14) even when the run ends sooner; `valid_until_step` caps it at the
  run end, and the sheet's "Ends" shows the real end (as for the exact-hold permission).
