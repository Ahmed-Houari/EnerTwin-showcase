# EnerTwin v3: the full-year experiment (2026-10-19)

How much electricity EnerTwin saves over a whole typical year, on the v3 building, in simulation. A separate experiment:
nothing in the AI, its guards, the validated results or the physics changed. `scripts/v3_year.py` only runs and reads;
the results are in `data/v3_year/summary.json` (local, untracked) and on the Energy page's **Year** view.

**Read this first.** Every number here is simulated by the v3 EnergyPlus model, which is not calibrated to a real
building; nothing is metered. Most days of the year (281 of 365) are reported days of the surrogate's training periods,
so this is an energy estimate over a typical year, not a test of the AI's accuracy (the locked test fortnights are that:
`docs/v3_results.md`).

## Headline (typical year, simulated)

| | S1 Conventional operation | S2 Smart vacancy control | S3 EnerTwin |
|---|---:|---:|---:|
| Electricity, kWh/yr | 300,981 | 287,874 | 268,281 |
| Lighting, kWh | 79,569 | 69,709 | 69,709 |
| Cooling coil, kWh | 108,512 | 106,250 | 95,294 |
| Supply fan, kWh | 31,294 | 30,263 | 21,648 |
| EUI, kWh/m²·yr | 126.7 | 121.2 | 112.9 |
| Highest peak, kW | 151.1 | 149.0 | 131.1 |
| Occupied zone-hours outside 19.5–26 °C | 0.0 | 0.0 | 0.0 |
| Peak at | 2001-08-05 14:55 | 2001-08-26 15:00 | 2001-08-26 14:55 |

Peak times are the start of the 5-minute step. "Occupied zone-hours outside 19.5–26 °C" is zero in all three: no occupied
zone left the comfort band in any scenario.

**Savings three ways:**

| Saving | kWh/yr | % | of which lighting, kWh | of which cooling coil, kWh |
|---|---:|---:|---:|---:|
| S1 → S2 (vacancy sensors) | 13,107 | 4.35 | 9,860 | 2,262 |
| S2 → S3 (the AI) | 19,593 | 6.81 | 0 | 10,956 |
| S1 → S3 (EnerTwin in total) | 32,700 | 10.86 | 9,860 | 13,219 |

- **S1 → S2, the vacancy sensors: 13,107 kWh a year (4.35 %).** Almost all lighting (9,860 kWh), plus 2,262 kWh of
  cooling-coil electricity because less light means less heat to remove.
- **S2 → S3, the AI: 19,593 kWh a year (6.81 % of S2).** All cooling side: 10,956 kWh at the cooling coil and 8,615 kWh
  at the supply fan (the VAV fan moves less air at a 25.5 °C hold). No lighting saving: see the protocol note
  below. The AI also lowers the year's highest peak from 149.0 to 131.1 kW. The verifications of the 252 executed
  actions, each against its own no-action replay, add up to 20,418 kWh; the segment totals (19,593 kWh) are the figure
  used here.
- **S1 → S3, EnerTwin in total: 32,700 kWh a year (10.86 %)**, EUI from 126.7 to 112.9 kWh/m²·yr.

The AI's share grows with the cooling load: 0.4–1.2 % of S2 in December–March, 9.2–10.4 % in June–September (table below).

**What the AI did (S3, 260 workdays it could act on):** 252 cooling holds executed, 242 of them the fixed rule's own
hold (25.5 °C 09:00–18:00), the rest 26.0 °C holds (4), later starts (4), one pre-cool and one hold to 17:00; no guard
stopped any, no suspension, AUTO not used. On 6 workdays (January, February, 23 December) the best proposal was the
5-minute lighting delay, which this protocol does not answer, so it expired (EnergyPlus had projected 19.6 kWh for the
six). On 2 workdays (22 February, 25 December) the proposed hold projected an energy *increase* (+0.04 and +4.0 kWh);
the protocol approves without acknowledging an increase, so the runtime did not execute them and they expired.
18 decisions used the screening fallback (input outside the surrogate's training range, all in January–April and
October–December); EnergyPlus confirmed every executed action as always.

**Reproducibility check:** the three locked test fortnights inside the year (`february_fortnight`, `spring_fortnight`,
`summer_fortnight`) reproduced the validated `manual_surrogate_v2` runs of `docs/v3_allseason.md` to 0.1 kWh (June:
14,156.9 kWh used against 15,678.3 kWh without actions, 1,565.7 kWh verified; March 6,996.5 / 7,116.2; February
6,850.9 / 6,862.2).

## The three scenarios

Same building model (Riyadh, 3 floors, 2,376 m², 15 thermal zones, 29 rooms), same weather, same simulated people,
same plugs, same HVAC equipment, sizing and schedule (07:00–18:00 on workdays) in all three. Only the controls differ.

| | Lighting | Cooling setpoints |
|---|---|---|
| **S1 Conventional operation** | Every room at the occupied level, 90 % of the lighting power density 9.69 W/m² (8.72 W/m²), **08:00–18:00 on every workday (Sunday–Thursday)**, whoever is in; at the 18 % essential level (1.74 W/m², the level every scenario keeps in an empty room) at all other times and all weekend (Friday–Saturday). Never 24/7. | The approved v3 schedule |
| **S2 Smart vacancy control** ("without the AI") | The room controller: a room is lit (90 %) while occupied and drops to the 18 % essential level 15 minutes after it empties. | The approved v3 schedule |
| **S3 EnerTwin** ("with the AI") | As S2, plus any lighting change the AI makes (none in this protocol, see below). | The all-season optimizer `surrogate_v2` with its safety-margin options (a) and (b); every cooling proposal approved as in the manual validation protocol; every guard active |

S2 is exactly the no-action replay the validated results compare against. S1 uses the room controller's own levels
(`lit_fraction` 0.9, `essential_lighting_fraction` 0.18), so S1 and S2 differ only in *when* a room is lit.

**The manual protocol answers no lighting proposal.** The validated protocol (`manual_surrogate_v2` in
`scripts/v3_validation_fortnights.py`) approves every HVAC proposal and leaves the rest unanswered. On days where the
optimizer's best choice was the 5-minute vacancy lighting delay, that proposal expired and the AI changed nothing that
day. Over the year that happened on 6 workdays (EnergyPlus had projected 19.6 kWh for them), so S3's lighting equals S2's and the AI's whole saving here is on the cooling side. The protocol was kept as validated rather than changed for this experiment.

## Weather: a typical year spliced from 12 source years

`data/weather/SAU_RI_Riyadh-Khalid.Intl.AP.404370_TMYx.2011-2025.epw` (TMYx, NCEI ISD/ERA5, period of record 2011–2025).
Like every typical-year file it joins months from different years, each chosen as typical for its calendar month:

| Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | Nov | Dec |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 2016 | 2024 | 2013 | 2013 | 2011 | 2020 | 2015 | 2015 | 2015 | 2014 | 2017 | 2016 |

That is standard for annual energy studies; it means month boundaries are joins of different real years, and the year
represents typical rather than any actual weather. The calendar is 2001 (1 January a Monday), as in every v3 run.

## Method: consecutive segments, not one continuous year

A demo run's decisions (the counterfactual) and verifications replay the run from its first day, so one run's cost
grows with the square of its length: one continuous AI year is not feasible. Every scenario therefore runs in the same
**28 consecutive segments**, each with the same warm-up as the validated fortnights (a 14-day no-action lead-in,
clamped at 1 January), and the three scenarios of a segment share it:

- `year_jan01` (1 January): S1 and S2 only. It is EnergyPlus's warm-up day; a forecast from it cannot replay the past
  exactly, so the runtime's replay check refuses it (the guard was not changed). Its S3 energy is its S2 run.
- `year_00` (2–4 January), then **25 Friday–Thursday fortnights** (5 January – 20 December), then `year_26`
  (21–31 December).
- The chunks are fortnights, not calendar months: a fortnight is the validated run length, three of the 25 are exactly
  the locked test fortnights (`february_fortnight`, `spring_fortnight`, `summer_fortnight`, simulated here as evaluation
  only, nothing trained; the fourth, `winter_fortnight` 14–27 December, falls across `year_25` and `year_26`), and a
  fortnight replays faster than a month. Results are summed per calendar month afterwards. (One continuous AI year was
  never run: segments were the plan from the start because of the quadratic replay.)
- Reference sizing: each segment's batch runs size the HVAC from the design days with the validated method
  (`energy_model_v3_occupancy.compare`: reference run, then occ_zone and occ_room on its sizing and People schedule);
  S1 reuses the same sizing and People schedule, only the lighting schedule differs.
- Lost at the segment boundaries: anything an earlier segment's AI change would carry into the next (thermal mass, a
  hold running past midnight) and the warm-up history. For S2 a continuous run measures the size of the
  boundaries (it cannot measure an AI change carried over; S3 has no continuous run): one continuous S2 run of the whole year (`comparison_annual.json`, the same inputs, no segments) used 288,125.7 kWh against 287,874.3 kWh summed over the segments: **−251.4 kWh (−0.09 %)**, small against every saving reported here. Its peak is the same 149.0 kW on 26 August.

One side effect outside the experiment: the runtime's `v3_shadow_runtime.period_for` now resolves every day of 2001 to
its year segment, so a demo or shadow run can start on any day that has the segment's batch results on the machine
(before, days inside the surrogate's training periods were refused). The `train_*` periods themselves are still never
demo periods (`tests/test_v3_surrogate.py`), and `annual` is not a demo period.

Counted per 5-minute step of each segment's reported days (365 × 288 steps per scenario): building electricity
(`Electricity:Facility`), lighting, cooling-coil electricity and supply-fan electricity; for S1 and S2 from each batch
run's EnergyPlus SQL, for S3 from the run's own steps. "Occupied zone-hours outside 19.5–26 °C": zone-steps with at
least one occupant whose air temperature is outside the band, ÷ 12.

## Month by month

kWh per month; savings in kWh with the percentage of the earlier scenario. "AI changes" are executed cooling holds.

| Month | S1 kWh | S2 kWh | S3 kWh | S1→S2 kWh (%) | S2→S3 kWh (%) | S1→S3 kWh (%) | AI changes | guard stops | suspensions |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Jan | 16,257 | 15,379 | 15,313 | 877 (5.4) | 66 (0.4) | 944 (5.8) | 19 | 0 | 0 |
| Feb | 14,604 | 13,830 | 13,761 | 774 (5.3) | 70 (0.5) | 843 (5.8) | 17 | 0 | 0 |
| Mar | 16,512 | 15,621 | 15,427 | 891 (5.4) | 194 (1.2) | 1,085 (6.6) | 21 | 0 | 0 |
| Apr | 19,101 | 18,028 | 17,582 | 1,074 (5.6) | 445 (2.5) | 1,519 (8.0) | 22 | 0 | 0 |
| May | 27,833 | 26,571 | 24,567 | 1,262 (4.5) | 2,004 (7.5) | 3,266 (11.7) | 23 | 0 | 0 |
| Jun | 32,880 | 31,694 | 28,764 | 1,186 (3.6) | 2,930 (9.2) | 4,116 (12.5) | 20 | 0 | 0 |
| Jul | 40,588 | 39,177 | 35,098 | 1,411 (3.5) | 4,079 (10.4) | 5,490 (13.5) | 23 | 0 | 0 |
| Aug | 40,972 | 39,600 | 35,537 | 1,372 (3.4) | 4,063 (10.3) | 5,435 (13.3) | 22 | 0 | 0 |
| Sep | 32,753 | 31,571 | 28,479 | 1,182 (3.6) | 3,093 (9.8) | 4,274 (13.1) | 21 | 0 | 0 |
| Oct | 26,346 | 25,127 | 23,066 | 1,219 (4.6) | 2,061 (8.2) | 3,280 (12.5) | 23 | 0 | 0 |
| Nov | 17,235 | 16,281 | 15,798 | 955 (5.5) | 482 (3.0) | 1,437 (8.3) | 21 | 0 | 0 |
| Dec | 15,900 | 14,996 | 14,889 | 905 (5.7) | 107 (0.7) | 1,011 (6.4) | 20 | 0 | 0 |
| **Year** | **300,981** | **287,874** | **268,281** | **13,107 (4.4)** | **19,593 (6.8)** | **32,700 (10.9)** | **252** | **0** | **0** |

### Lighting and cooling-coil electricity per month (kWh)

The rest of each total is plugs (the same in all three), supply fan and reheat.

| Month | Lighting S1 | Lighting S2 | Lighting S3 | Cooling S1 | Cooling S2 | Cooling S3 |
|---|---:|---:|---:|---:|---:|---:|
| Jan | 6,896 | 6,021 | 6,021 | 1,379 | 1,344 | 1,299 |
| Feb | 6,100 | 5,357 | 5,357 | 1,435 | 1,391 | 1,343 |
| Mar | 6,564 | 5,788 | 5,788 | 2,411 | 2,335 | 2,222 |
| Apr | 6,631 | 5,800 | 5,800 | 4,596 | 4,421 | 4,202 |
| May | 6,896 | 6,021 | 6,021 | 11,195 | 10,927 | 9,747 |
| Jun | 6,299 | 5,540 | 5,540 | 15,868 | 15,576 | 13,900 |
| Jul | 6,896 | 6,025 | 6,025 | 20,860 | 20,493 | 18,196 |
| Aug | 6,730 | 5,889 | 5,889 | 21,311 | 20,950 | 18,682 |
| Sep | 6,465 | 5,685 | 5,685 | 15,081 | 14,816 | 13,137 |
| Oct | 6,896 | 6,020 | 6,020 | 9,484 | 9,263 | 8,154 |
| Nov | 6,465 | 5,662 | 5,662 | 3,175 | 3,074 | 2,818 |
| Dec | 6,730 | 5,901 | 5,901 | 1,716 | 1,661 | 1,595 |
| **Year** | **79,569** | **69,709** | **69,709** | **108,512** | **106,250** | **95,294** |

### Segments

S2 is the segment's no-action batch run, S3 the AI's run (`tmp/v3_year/segments/<name>/`), S1 the conventional batch run
(`conventional_<name>.json`). "verified" is the sum of the segment's action verifications.

| Segment | First day | Days | S1 kWh | S2 kWh | S3 kWh | verified, kWh | fallbacks | run time, s |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| year_jan01 | 2001-01-01 | 1 | 629 | 590 | 590 | 0 | 0 | — |
| year_00 | 2001-01-02 | 3 | 1,819 | 1,708 | 1,704 | 3 | 1 | 51 |
| year_01 | 2001-01-05 | 14 | 7,164 | 6,767 | 6,736 | 31 | 1 | 373 |
| year_02 | 2001-01-19 | 14 | 7,267 | 6,894 | 6,860 | 33 | 2 | 882 |
| february_fortnight | 2001-02-02 | 14 | 7,217 | 6,862 | 6,851 | 11 | 1 | 747 |
| year_04 | 2001-02-16 | 14 | 7,407 | 6,991 | 6,926 | 81 | 1 | 549 |
| spring_fortnight | 2001-03-02 | 14 | 7,534 | 7,116 | 6,997 | 133 | 1 | 362 |
| year_06 | 2001-03-16 | 14 | 7,605 | 7,180 | 7,114 | 81 | 1 | 341 |
| year_07 | 2001-03-30 | 14 | 8,992 | 8,466 | 8,225 | 280 | 1 | 333 |
| year_08 | 2001-04-13 | 14 | 8,987 | 8,513 | 8,329 | 194 | 1 | 345 |
| year_09 | 2001-04-27 | 14 | 10,098 | 9,584 | 9,052 | 562 | 0 | 388 |
| year_10 | 2001-05-11 | 14 | 13,219 | 12,669 | 11,668 | 1,063 | 0 | 352 |
| year_11 | 2001-05-25 | 14 | 14,360 | 13,775 | 12,558 | 1,261 | 0 | 378 |
| summer_fortnight | 2001-06-08 | 14 | 16,268 | 15,678 | 14,157 | 1,566 | 0 | 400 |
| year_13 | 2001-06-22 | 14 | 15,994 | 15,395 | 13,881 | 1,563 | 0 | 376 |
| year_14 | 2001-07-06 | 14 | 18,222 | 17,597 | 15,803 | 1,850 | 0 | 364 |
| year_15 | 2001-07-20 | 14 | 18,194 | 17,580 | 15,774 | 1,865 | 0 | 359 |
| year_16 | 2001-08-03 | 14 | 18,212 | 17,593 | 15,795 | 1,851 | 0 | 356 |
| year_17 | 2001-08-17 | 14 | 18,900 | 18,281 | 16,372 | 1,963 | 0 | 341 |
| year_18 | 2001-08-31 | 14 | 15,876 | 15,317 | 13,832 | 1,535 | 0 | 337 |
| year_19 | 2001-09-14 | 14 | 15,118 | 14,549 | 13,111 | 1,486 | 0 | 342 |
| year_20 | 2001-09-28 | 14 | 12,697 | 12,151 | 11,095 | 1,109 | 2 | 341 |
| year_21 | 2001-10-12 | 14 | 11,479 | 10,947 | 10,097 | 900 | 1 | 338 |
| year_22 | 2001-10-26 | 14 | 9,789 | 9,295 | 8,757 | 584 | 1 | 339 |
| year_23 | 2001-11-09 | 14 | 8,096 | 7,646 | 7,428 | 231 | 1 | 331 |
| year_24 | 2001-11-23 | 14 | 7,419 | 6,999 | 6,914 | 97 | 1 | 307 |
| year_25 | 2001-12-07 | 14 | 6,994 | 6,596 | 6,569 | 29 | 1 | 278 |
| year_26 | 2001-12-21 | 11 | 5,428 | 5,134 | 5,089 | 59 | 1 | 543 |

## Sanity check: S1 lighting by hand

S1 definition, exactly: lighting power density 9.69 W/m² over 2,376 m² of lit floor (all 15 zones), so the full
installed load is 23.0234 kW. 90 % of it (20.72 kW) from 08:00 to 18:00 on Sunday–Thursday; 18 % (4.14 kW) at all other
times. No holidays in the occupancy calendar.

| | Hours | Fraction | kWh |
|---|---:|---:|---:|
| Working hours: 261 workdays × 10 h | 2,610 | 0.90 | 23.0234 × 0.90 × 2,610 = 54,082 |
| All other hours: 8,760 − 2,610 | 6,150 | 0.18 | 23.0234 × 0.18 × 6,150 = 25,487 |
| **Year, by hand** | 8,760 | | **79,569** |
| **Year, simulated (EnergyPlus `Zone Lights Electricity Energy`, all zones)** | | | **79,569** |

The simulated S1 lighting equals the hand calculation to the kWh (difference −0.0 kWh), so S1 is exactly the schedule it
claims to be. The first S1 day checked alone (1 January, a Monday) gave 265.2 kWh simulated and by hand
(23.0234 × (0.9 × 10 + 0.18 × 14)).

## What this shows and what it does not

- **Shown (in simulation):** over a typical TMYx year on this model, the size of the saving from vacancy lighting
  control over conventional operation, and the further saving from the AI's cooling holds, month by month, with every
  guard active and comfort counted.
- **Not shown:** a metered saving; a calibrated building; the AI's accuracy (281 of 365 days are reported days of the
  surrogate's training periods; the locked tests are in `docs/v3_results.md`); what an AUTO grant would do (AUTO was not
  used); any lighting saving from the AI (its lighting proposals are not answered in this protocol); effects across a
  segment boundary.

## Reproduce

```
python -m scripts.v3_year prepare --workers 12      # S2 and sizing (skips finished periods)
python -m scripts.v3_year run --parallel 4          # S3 (skips finished segments)
python -m scripts.v3_year conventional --workers 8  # S1 (skips finished periods)
python -m scripts.v3_year report                    # data/v3_year/summary.json
```

Every job waits for 1.5 GB of free memory and 6 GB of Windows commit headroom above its own need before it starts, and a
failed job resumes on the next call, skipping finished work. Run times on the development machine (32 threads): S2 batch runs and the continuous annual run (`prepare`) with 12 workers; the 27 AI segments about 55 minutes wall-clock with 4 in parallel (2.9 h of run time; 280–880 s each); the 27 S1 runs under 2 minutes with 8 workers; the report about 9 minutes (it reads every 5-minute step of 3 × 365 days).
