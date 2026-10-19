# EnerTwin v3: validated results (single source of truth, 2026-10-18; all-season v2 section added the same night)

Simulated control of a simulated building: a three-floor office modeled in EnergyPlus (v3), Riyadh weather (TMYx,
June 2020 source year), simulated occupancy. No meter, sensor or physical equipment is involved. Every figure below is
a model result.

## The four validated fortnights (8–21 June, the same 14 days, the same model)

Each was produced by the automated EnergyPlus test of the live demo runtime (interrupted on day 10 and resumed with an
exact replay, max diff 0.0) and **reproduced exactly on the final code of 2026-10-18** (every command's projection
and verification identical). Each action is verified against an independent no-action EnergyPlus replay over its own
horizon (its day and three more, then on to the end of the next workday), rebound included.

| Run | Optimizer | Approvals | Verified saving | Used | Share of the no-action fortnight |
|---|---|---|---|---|---|
| Manual | detection list (4–6 candidates) | 10 manual | **−1,072.9 kWh** | 14,627.1 kWh | 6.8 % |
| Exact-hold AUTO (one grant) | detection list | 0 | **−1,138.8 kWh** (holds −1,069.2, lighting −69.6) | 14,556.2 kWh | 7.3 % |
| Surrogate, manual | ML-screened grid, EnergyPlus confirms | 10 manual | **−1,565.7 kWh** | 14,156.9 kWh | 10.0 % |
| **Flexible AUTO (envelope permission), one grant** | ML-screened grid, EnergyPlus confirms | **0** | **−1,629.1 kWh** (holds −1,561.6, lighting −67.5) | **14,089.2 kWh** | **10.4 %** |

**No-action consumption of the fortnight:** 15,678.3 kWh (the EnergyPlus batch of the same 14 days with no action,
occ_room case, `data/synthetic/energy_model_v3_poc/occupancy/comparison_summer_fortnight.json`; the stepping runtime
matches it on all 4,032 steps). The envelope run's verified saving is **10.4 %** of it. Measured the other way, the
envelope run used 1,589.1 kWh (10.1 %) less than the no-action fortnight; the sum of the per-action verifications is
40 kWh larger because each action is measured against a replay without that action only, with the other actions kept,
and the last three days' horizons are cut by the end of the run (their later rebound is not simulated, in every run
above alike).

## What the envelope result does and does not show

- **The same action won every workday.** On all ten workdays EnergyPlus confirmed the surrogate's first pick, "hold
  cooling at 25.5 °C, 09:00–18:00", plus the 5-minute vacancy lighting delay; weekends were no action. A fixed rule
  "25.5 °C, 09:00–18:00 every workday" would have matched it on this fortnight (run since with the same harness:
  identical, verification for verification; see the March section). The surrogate's contribution is that it
  found that choice among 242 documented options at every decision without a hand-written rule; on days where the
  weather, the morning state or the rebound favour another window or a pre-cool it would pick that one (the training
  data contain such days), but this fortnight does not demonstrate it.
- **Comfort trade-off.** The saving comes from warmer occupied rooms, inside the permitted comfort band: during a hold the
  warmest occupied zone reaches 25.5 °C (exact-hold AUTO: 25.0 °C; no action: 24.0 °C). Zero occupied zone-hours fell
  outside 19.5–26 °C, and every verification's comfort and occupied-lighting checks passed. Whether 25.5 °C is
  acceptable for the occupants is a policy choice the model cannot make; the envelope permits 24.5–26.0 °C.
- **Peak.** The 25.5 °C hold cuts each workday's peak by about 21 kW (exact hold at 25 °C: about 15 kW).
- **One building, one climate model.** A second test fortnight (2–15 March) is below; 30 days have no approved
  period.

## All-season surrogate v2 against the fixed rule (milestone, 2026-10-18)

Full report: `docs/v3_allseason.md`. Test sets locked before any v2 data (`config/v3_test_periods.json`, commit
`21b6c43`): a new, untouched **primary test 14–27 December** (winter), plus June (validated) and March (seen once).
The v2 surrogate is trained on 20,092 EnergyPlus rows from all twelve months (the three test fortnights excluded), and
the v2 optimizer (`--optimizer surrogate_v2`, opt-in) has EnergyPlus confirm the fixed rule's 25.5 °C hold, the 26.0 °C
hold and the 5-minute lighting delay at every decision, screened or fallback.

Energy used over each fortnight (kWh; below no action in brackets):

| Period | No action | v2 AI manual | Fixed rule 25.5 °C 09–18 | v2 Flexible AUTO (0 approvals) | Fixed rule + lighting |
|---|---|---|---|---|---|
| **14–27 Dec (primary)** | 6,702.4 | 6,657.9 (44.5) | **6,653.0 (49.4)** | 6,608.5 (93.9) | **6,603.9 (98.5)** |
| 8–21 Jun | 15,678.3 | 14,156.9 (1,521.4) | 14,156.9 (1,521.4) | 14,089.2 (1,589.1) | 14,089.2 (1,589.1) |
| 2–15 Mar (seen once) | 7,116.2 | **6,991.6 (124.6)** | 6,996.5 (119.7) | 6,997.2 (119.0) | **6,945.8 (170.4)** |

- **The milestone "never worse than the fixed rule" is not met in verified energy.** June: equal, verification for
  verification. March: the manual AI beats the rule by 4.9 kWh, but with 0.17 occupied zone-hours at 26.000013 °C
  (one zone, 10 min, Tue 13 March, a 26 °C hold); with Flexible AUTO that overshoot tripped the live comfort guard, AUTO
  stayed suspended (by design) and the AI trails rule + lighting by 51.4 kWh. December: the AI trails by 4.9 kWh
  (manual) and 4.6 kWh (Flexible): on five cold days nothing reaches the 2 kWh / 0.5 % benefit floor, so the AI does
  not act; the rule acts and its small saving is real.
- **Projection level, proven:** in all 50 v2 decisions where the rule's own projection was beneficial, the chosen action
  was projected at least as good as the rule; on the three truth grids the v2 confirmed set reaches the true best on
  all 30 decisions (regret 0.000 kWh).
- **Seasons:** v2 falls back on 1 of 10 December workdays (v1: 10 of 10), and decides by season: 25.5 °C every June
  workday, 25.5 or 26.0 °C in March, 26.0 °C or no action in December.
- **June path unchanged:** re-run on the final code, the v1 surrogate runs reproduce −1,565.7 kWh (manual) and
  −1,629.1 kWh (Flexible AUTO) command for command; the detection-list manual run gives −1,072.9 kWh; the defaults
  (`energyplus` for the API, `surrogate` v1 on a double-click) are unchanged.

### Two safety-margin options of v2 and a fourth, untouched test (2026-10-19)

Adopted 2026-10-19: (a) a 26.0 °C hold only when the warmest occupied zone is projected at most 25.9 °C; (b) the fixed rule's
25.5 °C hold also below the 2 kWh / 0.5 % benefit floor when its projection still saves energy, keeps comfort and does
not raise the peak (`surrogate_v2`, run setting `v2_options`, on by default). New test locked first: **2–15 February**,
held out of a refit of the v2 recipe without February (`v3_surrogate_v2h`), since every month was in v2's training.

Energy used, kWh (lower is better):

| Period | v2 + options, manual | Fixed rule | v2 + options, Flexible AUTO | Fixed rule + lighting | No action |
|---|---|---|---|---|---|
| **2–15 Feb (untouched)** | 6,850.9 (trails 1.0) | 6,849.9 | 6,806.2 (trails 0.9) | 6,805.3 | 6,862.2 |
| 14–27 Dec | 6,653.0 (equal) | 6,653.0 | 6,665.3 (trails 61.4) | 6,603.9 | 6,702.4 |
| 8–21 Jun | 14,156.9 (equal) | 14,156.9 | 14,089.2 (equal) | 14,089.2 | 15,678.3 |
| 2–15 Mar | 6,996.5 (equal) | 6,996.5 | 6,945.8 (equal) | 6,945.8 | 7,116.2 |

- Without the options v2 trailed the rule in February by 5.6 kWh (manual and Flexible alike).
- With them the AI is in practice the fixed rule (three different choices in 40 workdays) and no longer exceeds 26 °C
  (0 occupied zone-hours outside the band in every run). February's 1.0 kWh: two days with the rule's projection exactly
  0.0 kWh, where the rule still saved 1.0 kWh together. December's Flexible 61.4 kWh: a below-floor rule hold tripped the
  early-read guard (5.1 kWh above the no-action forecast, limit 5.0, on a day warmer than forecast; it verified a saving),
  the permission stayed suspended, and the last four workdays expired. **"Never worse than the rule" is still not met
  exactly**; full analysis and the remaining design choices: `docs/v3_allseason.md` section 8.

## Second test fortnight: 2–15 March (pure test, 2026-10-18)

Nothing changed, retrained or tuned; the June table above is unchanged and its envelope run was re-run on the final
code with identical results. Full report: `docs/v3_validation_march.md`.

- **Why 2–15 March:** the mildest season in which workdays still need cooling (mean daily max 28.5 °C against 43.5 °C
  in June; no-action fortnight 7,116.2 kWh against 15,678.3 kWh), two whole Fri–Thu weeks, one TMYx source year (2013,
  no splice), and no simulated day shared with any training period (the earliest training lead-in starts 18 March).
  Same design-day sizing (identical HVAC sizes); batch checks pass; stepping matches batch on all 4,032 steps.

| 2–15 March | Approvals | Verified saving | Used | Below no action | Warmest occupied air |
|---|---|---|---|---|---|
| Manual, EnergyPlus optimizer | 10 | −72.5 kWh | 7,049.3 kWh | 66.9 kWh | 25.0 °C |
| Manual, surrogate optimizer (fallback on all 10 workdays) | 10 | −72.5 kWh | 7,049.3 kWh | 66.9 kWh | 25.0 °C |
| Flexible-permission AUTO (fallback on all 10) | 0 | −118.6 kWh (holds −67.2, lighting −51.4) | 7,001.9 kWh | 114.3 kWh | 25.0 °C |
| **Fixed rule, 25.5 °C 09:00–18:00 every workday** | rule | **−132.6 kWh** | 6,996.5 kWh | **119.7 kWh** | 25.5 °C |
| **Fixed rule + 5-min lighting delay** | rule | **−182.3 kWh** (holds −131.4, lighting −50.9) | 6,945.8 kWh | **170.4 kWh** | 25.5 °C |

No-action fortnight 7,116.2 kWh. 0 occupied zone-hours outside 19.5–26 °C in every run.

- **The AI does not beat the fixed rule.** In March the rule wins: 119.7 against 66.9 kWh (cooling only) and 170.4
  against 114.3 kWh (with the lighting delay, against Flexible AUTO). In June the rule matches the AI exactly: rule
  −1,565.7 kWh of 14,156.9 kWh (= surrogate manual), rule + lighting −1,629.1 kWh of 14,089.2 kWh (= Flexible AUTO).
  The rule's hold is the surrogate's June pick, so in June it is not an independent baseline; March is a fair test of
  both.
- **Why the AI lost in March:** every workday had inputs outside the surrogate's training range (morning ground-floor
  zone air 20.0–20.7 °C, on seven days also a cold 07:00 outdoor temperature), so it fell back to the detection list on
  all ten, as designed; that list offers only 25 °C holds. EnergyPlus's whole-grid truth says the best March choice is a
  09:00–18:00 hold at 25.5 or 26.0 °C, within 0–1.5 kWh a day of the rule. Flexible AUTO also lost Tue 13 March: the
  AUTO peak check refused a short afternoon hold ("raises the day peak"), which then expired without an operator.
- **Surrogate on March (truth grid, 2,430 EnergyPlus runs):** top-3 recall 1/10 (June 10/10), regret mean 1.18 kWh,
  kWh MAE 3.21 with R² 0.67 (June 0.96), rank correlation 0.894 (June 0.993). Outside its training range it ranks
  poorly, so the fallback was right.

## A full typical year in three scenarios (2026-10-19; an energy estimate, not a test)

Details, method and every table: `docs/v3_year.md`. The whole TMYx year (1 January – 31 December; months from 12 source
years, 2011–2025) on the same model, weather, people, plugs and HVAC schedule, in 28 consecutive segments (Fri–Thu
fortnights with the validated 14-day lead-in; one continuous AI run is infeasible because the replay cost grows with the
square of its length). Nothing in the AI, its guards or the results above changed.

| Typical year (simulated) | S1 Conventional operation | S2 Smart vacancy control (without the AI) | S3 EnerTwin (with the AI) |
|---|---:|---:|---:|
| Electricity, kWh/yr | 300,981 | 287,874 | 268,281 |
| EUI, kWh/m²·yr (2,376 m²) | 126.7 | 121.2 | 112.9 |
| Highest peak, kW | 151.1 | 149.0 | 131.1 |
| Occupied zone-hours outside 19.5–26 °C | 0 | 0 | 0 |

- **S1 → S2 (vacancy sensors): −13,107 kWh (−4.35 %)**, nearly all lighting. S1 = every room lit at 90 % of 9.69 W/m²
  08:00–18:00 Sunday–Thursday whoever is in, 18 % otherwise; its simulated lighting (79,569 kWh) equals the hand
  calculation 9.69 W/m² × 2,376 m² × (0.9 × 2,610 h + 0.18 × 6,150 h).
- **S2 → S3 (the AI): −19,593 kWh (−6.81 %)**, all cooling side (coil 10,956 kWh, supply fan 8,615 kWh); 0.4–1.2 % of
  S2 in December–March, 9.2–10.4 % in June–September. 252 cooling holds executed on 260 workdays (242 the fixed rule's
  own hold), no guard stop, no suspension, manual protocol (HVAC proposals approved; 6 lighting proposals expired
  unanswered, 2 holds projecting an increase not executed). AUTO not used.
- **S1 → S3 (EnerTwin in total): −32,700 kWh (−10.86 %).**
- Inside the year, the February, March and June test fortnights reproduced their validated v2 runs to 0.1 kWh. The
  segment boundaries cost −251 kWh (−0.09 %) against one continuous S2 year.
- Simulated, uncalibrated, and 281 of the 365 days are reported days of the surrogate's training periods: this
  estimates energy over a typical year; it does not test the AI's accuracy.

## The ML surrogate (screening only)

- **Training data:** **11,664 rows**, one per candidate EnergyPlus projection run, from **252 decisions** (each with its
  own no-action baseline run: **11,916 EnergyPlus decision runs** in total), plus **12 continuous context runs** (six
  training periods × two contexts) that produce the persisted pasts. 105 distinct calendar days in April, May, July,
  August, September and October (each in a no-action and a random-action context), never 8–21 June.
- **Held-out fortnight (8–21 June, the whole 242-option grid by EnergyPlus on the ten workdays):** top-3 recall 10/10,
  regret 0 kWh, rank correlation 0.993; kWh MAE 6.7 (R² 0.96). About 0.1 ms per candidate against about 11 s per
  EnergyPlus run.
- **Second test fortnight (2–15 March, outside the training range):** fallback on all ten workdays; top-3 recall 1/10,
  kWh R² 0.67 (see the March section).
- **Cross-validation, day-grouped (2026-10-18 re-run):** a calendar day now never sits on both sides of a fold (both
  contexts of a day share weather and occupancy). kWh MAE 2.53 (R² 0.994) for the chosen gradient boosting, against
  1.71 with the earlier grouping, which was optimistic. With the new grouping extra trees would edge out gradient
  boosting for the outside-comfort-hours target (MAE 0.65 against 0.69 h); the validated artifact was kept, since the
  held-out evaluation, not the cross-validation, is the evidence and a refit would invalidate the validated runs.
- **Artifact:** `data/surrogate_v3/model/v3_surrogate_v1.joblib`, **39.6 MB** (was 119.6 MB), sha256 `5d40ce76…`,
  pinned in `scripts/v3_surrogate.py` and checked before it is unpickled. Training-only tree statistics were removed
  and it is xz-compressed; its predictions are bit-identical to the trained model's on all 11,664 rows and on a fixed
  probe (prediction fingerprint `45d8d6dc…`, also pinned). The original is archived beside it. Rebuild from the
  dataset: `python -m scripts.v3_surrogate_train --refit` reproduces the pinned prediction fingerprint exactly (the file
  bytes differ slightly; the predictions do not).

## Live runs (browser, Demo speed; not the validated table)

- **Fresh Flexible AUTO run on the final code (2026-10-18, the default view):** through the real web API at Demo
  speed, one grant on Friday 00:40, 20 AUTO actions, 0 manual approvals: **−1,629.1 kWh of 14,089.2 kWh**, identical to
  the validated fortnight (`docs/v3_demo_video.md`).
- Earlier live runs differ from the table only by when the operator acted: the surrogate live run with one manual
  approval before the grant (−1,623.6 kWh), the exact-hold live run with a revocation and re-grant (−1,060.3 kWh), the
  manual week (−560.8 kWh over 7 days). They are archived under `data/runtime_v3_demo/archive/` (never deleted).
