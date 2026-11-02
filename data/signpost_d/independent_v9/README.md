# independent_v9 — R32 targeted training records (`train_v6.jsonl`)

The last batch of training records in pilot 2's design, built as R32 asks with the same pipeline as train_v4 and train_v5. It targets four failure patterns (T1–T4) and asks for sharper clear records (T5).

- **Size:** 813 records in 40 new families, 4065 questions, ids `F401-0001` onward.
- **Format:** `train_v6.jsonl` is in System One format, one record per line, with the same fields as train_v5 plus `meta.targets` (which of T1–T5 each question serves).
- **Rules:** everything is invented and the R22.1 exclusions apply. No benchmark was opened, read or used. No family, question set, shared scale, record or seed is shared with pilot 1 (`independent_v6/pilot_v3.jsonl`, evaluation only), pilot 2 (`independent_v6/pilot_v3b.jsonl`) train_v4 or train_v5. The 40 domains are new: none of the 96 used before.

**Checks:** 72 of 119 R22.4.3 / R22.9 checks pass, overall and per family. The misses are listed under *Where the set misses the targets*.

**API cost:** about **$28.78** on the team's API key, including every dropped record and retry (cap $42).

## Files

| File | Purpose |
|---|---|
| `train_v6.jsonl` | the records: `id`, `family`, `domain`, `state`, `questions`, `gold`, `meta` (`factors`, `skills`, `plan`, `judgments`, `judge_setup`, `contested`, `deciding`, `off_plan`, `targets`) |
| `stats.json` | every R22.4.3 and R22.9 check, overall and per family, plus generation counts and API cost by model and stage |
| `qc/families.json` | the 40 family specifications: role, setting, question set, latent factors, sections, rendering notes, skills |
| `qc/shared_scales.json` | the three cross-family shared scales, new for this set |
| `qc/work/` | everything the run produced: families, every record written (kept or not) with its sampled and final factors, plan and deciding evidence, rejected records with the reason, every judgment, the batch results and the cost ledger |
| `qc/*.log` | stage logs from the worker sessions |
| `pipeline/` | the code |

`meta.deciding` is the writer's statement, per question, of the field or fact that settles it (or that holds a balanced or contested question in balance). `meta.off_plan` lists any question whose measured gold missed its plan.

## How it was made (the R26/R27 pipeline, unchanged except the plans)

All calls went through the Anthropic API, billed to the team's API key, from separate worker sessions. They used the Python SDK and structured outputs. Every request except the family designs went through the **Message Batches API** (half price).

1. **Shared scales:** Claude Opus 5.5, one call. Three new generic 4-level scales, each rating one quantity with checkable anchors:
   - `reversal_time`: How long would it take to fully undo this outcome? Levels: Could be fully undone within one day / Needs one to seven days to undo / Needs eight days to three months to undo / Cannot be undone within three months
   - `parties_needed`: How many separate people or teams must act to resolve this? Levels: One person can resolve it alone / Two people or teams must act / Three or four people or teams must act / Five or more people or teams must act
   - `staff_effort`: How much staff time would fully resolving this record require? Levels: Under one hour of staff time needed / One to eight hours of staff time / One to five working days of staff time / More than five working days of staff time
2. **Families:** Claude Opus 5.5, one direct call per family, up to 3 attempts.
   - **Slots:** each of the 40 slots copies the structure of one pilot-2 slot, so the set keeps pilot 2's mix: type mix, option and level counts, bare yes/no question, shared-scale use and skills. Each slot has a new domain.
   - **Design rules:** the designer follows pilot 2's anchoring rules. Every score question rates one quantity, each level is a checkable condition, and choice options differ along one axis.
   - **Checks:** code checks every R22.5 rule and the R22.1 exclusions.
3. **Plans:** in code, with train_v5's shares (balanced and contested plans as in R31), plus the R32 targets:
   - **T1, act when the record allows it:** in 24 of the 40 families, the first choice question is the next-step decision. Its options run from proceed, through request a specific missing item and act, to escalate. Half of the plans that land on "request information" are moved to another option, and the writer is told that answer is right only when a specific item is missing.
   - **T2, routine records:** 25% of records are planned routine, with the first choice option and the lowest score level, clear or leaning.
   - **T3, aggravating evidence:** in 16 of the 40 families, one boolean factor is a decisive aggravating fact. It is set in 30% of records, and in those records the main severity, risk or urgency question is planned at its top level.
   - **T4:** planned-true statements must carry the plain, ordinary evidence such a record holds.
   - **T5:** the writer is told to aim for about 0.9 on clear plans, with one decisive field.
   - **Tracking:** `meta.targets` lists the targets each question serves.
4. **Records:** Claude Sonnet 5.5, effort medium, **five records of one family per request**.
   - **Prompt:** the writing rules and the family brief are a **cached system prompt**. The writer returns each record, the final factor values it used, and the deciding field per question.
   - **Code checks:** 3–6 top-level keys, all from the family's sections, and at most 200 words.
   - **No rework:** a failing record is dropped. Requests that fail at the API are resubmitted once.
5. **Gold:** **three Claude Haiku 5.5 judgments per record**, each a separate batch request with no shared context. Judges see only the record, the role and the question set, and each gives a full distribution for every question. Sampling temperature cannot be set on these models, so the three samples differ by stance instead:

| Judge | Model | Effort | Stance |
|---|---|---|---|
| careful_operator | Claude Haiku 5.5 | medium | You are an experienced practitioner in this role: thorough, practical, and used to acting on records like this every day. |
| by_the_book | Claude Haiku 5.5 | medium | You are a by-the-book auditor: you apply the stated rules and thresholds literally and treat anything not on the record as unknown. |
| generalist | Claude Haiku 5.5 | medium | You are a sensible generalist new to this team: you read the record plainly and judge as an informed outsider would. |

   The gold is the plain mean of the three, rounded to 4 decimals, never sharpened or smoothed. All three judgments are in `meta.judgments`.
6. **Accept rule:** a question is off plan when its measured top answer is neither the planned answer nor, for a balanced or contested plan, the planned second answer. A record is kept when **at most one** of its five questions is off plan.
   - **Why one miss:** as in train_v4. Requiring every question to match keeps only about one record in five.
7. **Selection:** from the accepted records, 25 per family were chosen greedily so the set's measured statistics come as close as possible to R22.4.3, with no near-duplicate pair within a family. Gold values were never changed.

### Generation counts

- **Written:**
  - 2000 records written in total (per-family counts in `stats.json`, `generation.records_written_per_family`);
  - 50 records per family. There was no top-up: the first pass kept at least 75% of the target, so the top-up was skipped as instructed.
- **Dropped:**
  - 72 records failed the code checks (top-level keys outside the family sections: 29; more than 6 top-level sections: 20; over 200 words: 17; writer left out a record: 6);
  - 988 records failed the accept rule.
- **Kept:** 813.
- **Passed every check but not needed** (their family already had 25 closer to the targets): 127.
- **Families below 25 records:** F402 (7), F404 (15), F407 (14), F408 (20), F409 (15), F410 (7), F412 (19), F413 (19), F417 (12), F418 (14), F419 (19), F421 (23), F424 (21), F426 (17), F427 (24), F429 (22), F431 (20), F433 (12), F434 (11), F438 (20), F439 (11), F440 (21).

## Families

| Family | Domain | Skills | Questions | Records |
|---|---|---|---|---|
| F401 | hospital discharge readiness reviews | S1, S2, S3, S4, S5, S6, S7 | `rx_matches` (noul); `next_step` (choice); `destination` (choice); `outstanding_tasks` (score); `discharge_delay` (score) | 25 |
| F402 | community physiotherapy referral reviews | S1, S3, S4, S5, S6, S7 | `red_flag_unresolved` (noul); `next_step` (choice); `pathway` (choice); `reversal_time` (score); `clinical_urgency` (score) | 7 |
| F403 | school meal allergen plans | S2, S3, S4, S5, S6, S7 | `evidence_allergens_covered` (noul); `next_step` (choice); `first_gap_stage` (choice); `exposure_proximity` (score); `meal_urgency` (score) | 25 |
| F404 | nursery staffing ratio checks | S1, S3, S4, S5, S6, S7 | `below_ratio` (noul); `disposition` (choice); `primary_cause` (choice); `shortfall_duration` (score); `breach_severity` (score) | 15 |
| F405 | university timetable room clash reviews | S1, S2, S4, S5, S6, S7 | `genuine_clash` (noul); `alt_room_fits` (noul); `resolution` (choice); `urgency` (score); `parties_needed` (score) | 25 |
| F406 | museum visitor capacity logs | S1, S3, S4, S5, S6, S7 | `variation_applies` (noul); `fault_explains_overage` (noul); `next_step` (choice); `peak_excess` (score); `intervals_over` (score) | 25 |
| F407 | council pothole repair reports | S2, S3, S4, S5, S6, S7 | `meets_threshold` (noul); `repeat_failure` (noul); `next_step` (choice); `staff_effort` (score); `response_timescale` (score) | 14 |
| F408 | missed bin collection reports | S1, S3, S4, S5, S6, S7 | `crew_missed` (noul); `report_matches` (noul); `next_step` (choice); `return_urgency` (score); `repeat_failures` (score) | 20 |
| F409 | residents' parking permit applications | S1, S2, S3, S4, S5, S6, S7 | `residency_shown` (noul); `disposition` (choice); `main_issue` (choice); `misuse_strength` (score); `document_gaps` (score) | 15 |
| F410 | street market trader licence renewals | S1, S3, S4, S5, S6, S7 | `pli_term_covered` (noul); `renewal_disposition` (choice); `inspection_level` (choice); `reversal_time` (score); `discrepancy_count` (score) | 7 |
| F411 | taxi licence vehicle inspections | S2, S3, S4, S5, S6, S7 | `odometer_consistent` (noul); `next_step` (choice); `inspection_outcome` (choice); `document_gaps` (score); `outstanding_severity` (score) | 25 |
| F412 | food hygiene inspection follow-ups | S1, S3, S4, S5, S6, S7 | `all_remedied` (noul); `next_step` (choice); `worst_hazard` (choice); `overdue_extent` (score); `residual_risk` (score) | 19 |
| F413 | pest control visit logs | S1, S2, S4, S5, S6, S7 | `usage_reconciled` (noul); `activity_increased` (noul); `next_step` (choice); `breach_severity` (score); `parties_needed` (score) | 19 |
| F414 | playground equipment safety inspections | S1, S3, S4, S5, S6, S7 | `prior_defects_cleared` (noul); `surfacing_adequate` (noul); `asset_disposition` (choice); `remedial_progress` (score); `hazard_severity` (score) | 25 |
| F415 | public toilet cleaning logs | S2, S3, S4, S5, S6, S7 | `missed_clean` (noul); `sign_off_valid` (noul); `next_step` (choice); `staff_effort` (score); `fault_exposure` (score) | 25 |
| F416 | coastal lifeguard station logs | S1, S3, S4, S5, S6, S7 | `two_guard_cover` (noul); `totals_reconcile` (noul); `next_step` (choice); `outcome_severity` (score); `kit_outage` (score) | 25 |
| F417 | harbour mooring allocation records | S1, S2, S3, S4, S5, S6, S7 | `length_fits` (noul); `next_step` (choice); `primary_issue` (choice); `ukc_band` (score); `unresolved_count` (score) | 12 |
| F418 | rail station platform staffing logs | S1, S3, S4, S5, S6, S7 | `headcount_verified` (noul); `next_step` (choice); `coverage_outcome` (choice); `reversal_time` (score); `exposure_severity` (score) | 14 |
| F419 | airport runway friction test records | S2, S3, S4, S5, S6, S7 | `genuine_deficit` (noul); `next_step` (choice); `deficit_driver` (choice); `friction_severity` (score); `evidence_completeness` (score) | 19 |
| F420 | cargo pallet loading plans | S1, S3, S4, S5, S6, S7 | `pieces_reconciled` (noul); `disposition` (choice); `worst_discrepancy` (choice); `weight_deviation` (score); `rework_extent` (score) | 25 |
| F421 | parcel depot damaged-item records | S1, S2, S4, S5, S6, S7 | `depot_fault` (noul); `contents_match` (noul); `next_step` (choice); `loss_share` (score); `parties_needed` (score) | 23 |
| F422 | pharmaceutical warehouse returns quarantine | S1, S3, S4, S5, S6, S7 | `storage_maintained` (noul); `batch_traceable` (noul); `next_step` (choice); `resaleable_share` (score); `disposition_urgency` (score) | 25 |
| F423 | printing plant press-check records | S2, S3, S4, S5, S6, S7 | `colour_in_tolerance` (noul); `run_before_signoff` (noul); `next_step` (choice); `staff_effort` (score); `scrap_exposure` (score) | 25 |
| F424 | brewery batch fermentation logs | S1, S3, S4, S5, S6, S7 | `terminal_gravity` (noul); `contamination_supported` (noul); `batch_disposition` (choice); `deviation_severity` (score); `record_discrepancies` (score) | 21 |
| F425 | egg packing station grading records | S1, S2, S3, S4, S5, S6, S7 | `counts_reconcile` (noul); `batch_disposition` (choice); `anomaly_cause` (choice); `lay_to_pack` (score); `outgrade_share` (score) | 25 |
| F426 | orchard spray application records | S1, S3, S4, S5, S6, S7 | `phi_cleared` (noul); `next_step` (choice); `finding_class` (choice); `reversal_time` (score); `rate_overrun` (score) | 17 |
| F427 | peatland restoration monitoring visits | S2, S3, S4, S5, S6, S7 | `dam_caused_drop` (noul); `next_step` (choice); `hydro_state` (choice); `dam_severity` (score); `evidence_completeness` (score) | 24 |
| F428 | flood defence gate inspections | S1, S3, S4, S5, S6, S7 | `closure_test_valid` (noul); `next_step` (choice); `defect_class` (choice); `section_discrepancies` (score); `flood_risk` (score) | 25 |
| F429 | dam spillway monitoring logs | S1, S2, S4, S5, S6, S7 | `within_licence` (noul); `gates_reconciled` (noul); `next_step` (choice); `trigger_tier` (score); `parties_needed` (score) | 22 |
| F430 | electricity substation inspection rounds | S1, S3, S4, S5, S6, S7 | `defect_corroborated` (noul); `benign_explanation` (noul); `follow_up` (choice); `limit_deviation` (score); `supply_exposure` (score) | 25 |
| F431 | heat network meter readings | S2, S3, S4, S5, S6, S7 | `meter_fault` (noul); `billable_actual` (noul); `next_step` (choice); `staff_effort` (score); `consumption_deviation` (score) | 20 |
| F432 | zoo enclosure keeper logs | S1, S3, S4, S5, S6, S7 | `count_reconciles` (noul); `secure_at_close` (noul); `next_step` (choice); `intake_shortfall` (score); `missed_rounds` (score) | 25 |
| F433 | equestrian event vet checks | S1, S2, S3, S4, S5, S6, S7 | `pulse_met` (noul); `next_step` (choice); `outcome_code` (choice); `welfare_risk` (score); `record_completeness` (score) | 12 |
| F434 | cycling event marshal reports | S1, S3, S4, S5, S6, S7 | `coverage_held` (noul); `next_step` (choice); `incident_class` (choice); `reversal_time` (score); `consequence_severity` (score) | 11 |
| F435 | concert venue crowd capacity checks | S2, S3, S4, S5, S6, S7 | `verified_exceedance` (noul); `disposition` (choice); `discrepancy_source` (choice); `steward_shortfall` (score); `unconfirmed_exits` (score) | 25 |
| F436 | newspaper corrections desk reviews | S1, S3, S4, S5, S6, S7 | `error_confirmed` (noul); `next_step` (choice); `error_origin` (choice); `spread` (score); `harm_severity` (score) | 25 |
| F437 | translation agency job quality checks | S1, S2, S4, S5, S6, S7 | `scope_matches` (noul); `reviewed_in_time` (noul); `next_step` (choice); `open_severity` (score); `parties_needed` (score) | 25 |
| F438 | clinical coding audit samples | S1, S3, S4, S5, S6, S7 | `primary_supported` (noul); `hrg_shift` (noul); `next_step` (choice); `error_count` (score); `error_severity` (score) | 20 |
| F439 | driving test centre examiner reports | S2, S3, S4, S5, S6, S7 | `result_supported` (noul); `report_complete` (noul); `disposition` (choice); `staff_effort` (score); `issue_severity` (score) | 11 |
| F440 | youth hostel booking reviews | S1, S3, S4, S5, S6, S7 | `ratio_met` (noul); `deposit_covered` (noul); `disposition` (choice); `urgency` (score); `conflict_count` (score) | 21 |

- **Skill shares of families:** {"S1": 0.75, "S2": 0.5, "S3": 0.875, "S4": 1.0, "S5": 1.0, "S6": 1.0, "S7": 1.0}.
- **Domains:** 40.

## Label shape (R22.4.3)

| Type | Questions | top < 0.5 | 0.5–0.7 | 0.7–0.9 | ≥ 0.9 | mean top | train_v5 mean top | mean entropy |
|---|---|---|---|---|---|---|---|---|
| noul | 1254 | 0% | 19% | 36% | 45% | 0.829 | 0.805 | 0.396 |
| choice | 1185 | 22% | 45% | 27% | 5% | 0.631 | 0.6 | 0.951 |
| score | 1626 | 17% | 34% | 37% | 12% | 0.686 | 0.651 | 0.823 |

- **Contested share** (the three judges' top answers disagree): 16%.
- **Top-answer position:** score {"lowest": 0.299, "middle": 0.512, "highest": 0.189}; choice {"first": 0.206, "middle": 0.554, "last": 0.24}.
- **True/false:** `true` is top in 51%; `criteria: null` in 35%.
- **Score mass two or more levels from the top:** 0.0757 on average.
- **Gold zeros:** 0.
- **Records:** median 103 words (p10 61, p90 159, maximum 199).
- **Near-duplicate pairs within a family:** 0.

## R32 targets (measured)

| Measure | Value |
|---|---|
| T1_families | 24 |
| T1_next_step_questions | 496 |
| T1_request_info_top_share | 0.238 |
| T2_routine_records | 151 |
| T2_routine_share | 0.186 |
| T3_families | 16 |
| T3_planned_top_questions | 70 |
| T3_measured_top_share | 0.843 |
| T4_planned_true_questions | 753 |
| T4_measured_true_share | 0.757 |
| T5_clear_choice_questions | 442 |
| T5_clear_choice_mean_top | 0.643 |
| T5_clear_choice_share_ge_0_9 | 0.057 |

## Every check

| Check | Value | Target | Result |
|---|---|---|---|
| noul share top <0.5 | 0.0 | [0, 0] | pass |
| noul share top 0.5-0.7 | 0.19 | [0.3, 0.4] | **miss** |
| noul share top 0.7-0.9 | 0.364 | [0.28, 0.38] | pass |
| noul share top >=0.9 | 0.447 | [0.27, 0.37] | **miss** |
| noul mean top | 0.829 | [0.75, 0.81] | **miss** |
| noul mean entropy | 0.396 | [0.4, 0.5] | **miss** |
| choice share top <0.5 | 0.225 | [0.15, 0.25] | pass |
| choice share top 0.5-0.7 | 0.449 | [0.38, 0.5] | pass |
| choice share top 0.7-0.9 | 0.273 | [0.17, 0.27] | **miss** |
| choice share top >=0.9 | 0.052 | [0.1, 0.18] | **miss** |
| choice mean top | 0.631 | [0.62, 0.68] | pass |
| choice mean entropy | 0.951 | [0.75, 0.92] | **miss** |
| score share top <0.5 | 0.174 | [0.18, 0.28] | **miss** |
| score share top 0.5-0.7 | 0.338 | [0.45, 0.57] | **miss** |
| score share top 0.7-0.9 | 0.37 | [0.15, 0.25] | **miss** |
| score share top >=0.9 | 0.118 | [0.03, 0.09] | **miss** |
| score mean top | 0.686 | [0.58, 0.64] | **miss** |
| score mean entropy | 0.823 | [0.83, 0.98] | **miss** |
| contested share (judges' tops disagree) | 0.162 | [0.3, 0.4] | **miss** |
| score top at lowest | 0.299 | [0.1, 0.16] | **miss** |
| score top at middle | 0.512 | [0.57, 0.67] | **miss** |
| score top at highest | 0.189 | [0.22, 0.3] | **miss** |
| choice top at first | 0.206 | [0.15, 0.25] | pass |
| choice top at middle | 0.554 | [0.6, 0.7] | **miss** |
| choice top at last | 0.24 | [0.1, 0.18] | **miss** |
| noul true top | 0.506 | [0.45, 0.5] | **miss** |
| noul criteria null | 0.352 | [0.3, 0.36] | pass |
| score mass >= 2 levels from top (mean) | 0.0757 | [0, 0.1] | pass |
| F401 contested share | 0.2 | [0.15, 0.55] | pass |
| F401 noul true top | 0.32 | [0.25, 0.7] | pass |
| F402 contested share | 0.114 | [0.15, 0.55] | **miss** |
| F402 noul true top | 1.0 | [0.25, 0.7] | **miss** |
| F403 contested share | 0.128 | [0.15, 0.55] | **miss** |
| F403 noul true top | 0.32 | [0.25, 0.7] | pass |
| F404 contested share | 0.2 | [0.15, 0.55] | pass |
| F404 noul true top | 0.467 | [0.25, 0.7] | pass |
| F405 contested share | 0.144 | [0.15, 0.55] | **miss** |
| F405 noul true top | 0.64 | [0.25, 0.7] | pass |
| F406 contested share | 0.192 | [0.15, 0.55] | pass |
| F406 noul true top | 0.54 | [0.25, 0.7] | pass |
| F407 contested share | 0.214 | [0.15, 0.55] | pass |
| F407 noul true top | 0.536 | [0.25, 0.7] | pass |
| F408 contested share | 0.22 | [0.15, 0.55] | pass |
| F408 noul true top | 0.45 | [0.25, 0.7] | pass |
| F409 contested share | 0.04 | [0.15, 0.55] | **miss** |
| F409 noul true top | 0.933 | [0.25, 0.7] | **miss** |
| F410 contested share | 0.314 | [0.15, 0.55] | pass |
| F410 noul true top | 0.429 | [0.25, 0.7] | pass |
| F411 contested share | 0.104 | [0.15, 0.55] | **miss** |
| F411 noul true top | 0.56 | [0.25, 0.7] | pass |
| F412 contested share | 0.084 | [0.15, 0.55] | **miss** |
| F412 noul true top | 0.211 | [0.25, 0.7] | **miss** |
| F413 contested share | 0.095 | [0.15, 0.55] | **miss** |
| F413 noul true top | 0.368 | [0.25, 0.7] | pass |
| F414 contested share | 0.104 | [0.15, 0.55] | **miss** |
| F414 noul true top | 0.48 | [0.25, 0.7] | pass |
| F415 contested share | 0.144 | [0.15, 0.55] | **miss** |
| F415 noul true top | 0.64 | [0.25, 0.7] | pass |
| F416 contested share | 0.256 | [0.15, 0.55] | pass |
| F416 noul true top | 0.56 | [0.25, 0.7] | pass |
| F417 contested share | 0.133 | [0.15, 0.55] | **miss** |
| F417 noul true top | 0.167 | [0.25, 0.7] | **miss** |
| F418 contested share | 0.214 | [0.15, 0.55] | pass |
| F418 noul true top | 0.357 | [0.25, 0.7] | pass |
| F419 contested share | 0.168 | [0.15, 0.55] | pass |
| F419 noul true top | 0.474 | [0.25, 0.7] | pass |
| F420 contested share | 0.104 | [0.15, 0.55] | **miss** |
| F420 noul true top | 0.68 | [0.25, 0.7] | pass |
| F421 contested share | 0.191 | [0.15, 0.55] | pass |
| F421 noul true top | 0.522 | [0.25, 0.7] | pass |
| F422 contested share | 0.192 | [0.15, 0.55] | pass |
| F422 noul true top | 0.4 | [0.25, 0.7] | pass |
| F423 contested share | 0.24 | [0.15, 0.55] | pass |
| F423 noul true top | 0.56 | [0.25, 0.7] | pass |
| F424 contested share | 0.105 | [0.15, 0.55] | **miss** |
| F424 noul true top | 0.357 | [0.25, 0.7] | pass |
| F425 contested share | 0.192 | [0.15, 0.55] | pass |
| F425 noul true top | 0.24 | [0.25, 0.7] | **miss** |
| F426 contested share | 0.129 | [0.15, 0.55] | **miss** |
| F426 noul true top | 0.882 | [0.25, 0.7] | **miss** |
| F427 contested share | 0.158 | [0.15, 0.55] | pass |
| F427 noul true top | 0.292 | [0.25, 0.7] | pass |
| F428 contested share | 0.16 | [0.15, 0.55] | pass |
| F428 noul true top | 0.28 | [0.25, 0.7] | pass |
| F429 contested share | 0.127 | [0.15, 0.55] | **miss** |
| F429 noul true top | 0.568 | [0.25, 0.7] | pass |
| F430 contested share | 0.112 | [0.15, 0.55] | **miss** |
| F430 noul true top | 0.5 | [0.25, 0.7] | pass |
| F431 contested share | 0.25 | [0.15, 0.55] | pass |
| F431 noul true top | 0.45 | [0.25, 0.7] | pass |
| F432 contested share | 0.16 | [0.15, 0.55] | pass |
| F432 noul true top | 0.72 | [0.25, 0.7] | **miss** |
| F433 contested share | 0.183 | [0.15, 0.55] | pass |
| F433 noul true top | 0.583 | [0.25, 0.7] | pass |
| F434 contested share | 0.109 | [0.15, 0.55] | **miss** |
| F434 noul true top | 0.727 | [0.25, 0.7] | **miss** |
| F435 contested share | 0.144 | [0.15, 0.55] | **miss** |
| F435 noul true top | 0.36 | [0.25, 0.7] | pass |
| F436 contested share | 0.136 | [0.15, 0.55] | **miss** |
| F436 noul true top | 0.4 | [0.25, 0.7] | pass |
| F437 contested share | 0.2 | [0.15, 0.55] | pass |
| F437 noul true top | 0.46 | [0.25, 0.7] | pass |
| F438 contested share | 0.17 | [0.15, 0.55] | pass |
| F438 noul true top | 0.575 | [0.25, 0.7] | pass |
| F439 contested share | 0.127 | [0.15, 0.55] | **miss** |
| F439 noul true top | 0.545 | [0.25, 0.7] | pass |
| F440 contested share | 0.248 | [0.15, 0.55] | pass |
| F440 noul true top | 0.667 | [0.25, 0.7] | pass |
| every family has exactly 5 questions | True | True | pass |
| records have 3-6 sections | [3, 6] | [3, 6] | pass |
| record words max <= 320 | 199 | [0, 320] | pass |
| skill S1 share of families | 0.75 | [0.4, 1] | pass |
| skill S2 share of families | 0.5 | [0.25, 1] | pass |
| skill S3 share of families | 0.875 | [0.5, 1] | pass |
| at least 6 domains | 40 | [6, null] | pass |
| no near-duplicate records in a family (>50% of field values equal) | 0 | 0 | pass |
| T1 request-info top share on next-step questions | 0.238 | [0, 0.25] | pass |
| T1 families share | 0.6 | [0.6, 1] | pass |
| T3 families share | 0.4 | [0.4, 1] | pass |

## Where the set misses the targets

47 of 119 checks miss: 20 overall and 27 per family (19 contested share, 8 `true` top share). Many of these misses come from the R32 targets themselves, which deliberately add clear, low-end and top-level records.

- **No top-up, as instructed:**
  - 813 records were kept from 2,000 written, which is within 25% of the 1,000 target, so the top-up was skipped.
  - 18 families have 25 records and 22 have fewer: F402 (7), F404 (15), F407 (14), F408 (20), F409 (15), F410 (7), F412 (19), F413 (19), F417 (12), F418 (14), F419 (19), F421 (23), F424 (21), F426 (17), F427 (24), F429 (22), F431 (20), F433 (12), F434 (11), F438 (20), F439 (11), F440 (21).
  - The accept rule dropped more than in earlier batches (988 records).
- **Contested share 16%** against 30–40% (train_v5 21%). Routine (T2) and aggravated (T3) records are clear by design, and three stances of one model seldom disagree.
- **The low end is now over-used:**
  - The lowest score level is top on 30% of score questions (R22.4.3 10–16%; R32 asked for "at least" that share).
  - Score middle levels are top on 51% (target 57–67%) and the highest on 19% (target 22–30%).
  - The last choice option is top on 24% (target 10–18%), mostly escalation on aggravated records. The first choice option is top on 21%, inside the target.
  - Routine records make up 19% of the kept records (planned 25%).
- **Score gold is sharper than R22.4.3:** mean top 0.686 against 0.58–0.64. 37% of score questions sit at 0.7–0.9 and 12% at ≥ 0.9, mostly the clear routine and aggravated records.
- **T5 only partly met:** clear choice plans reach a mean top of 0.64, and only 5% of choice questions are at ≥ 0.9 (target 10–18%). A single Haiku judge rarely gives 0.9 to one of four or five options; this is the judges' ceiling rather than the writer's.
- **True/false is sharper:**
  - mean top 0.829 against 0.75–0.81;
  - 45% of questions at ≥ 0.9 against 27–37%;
  - `true` top in 50.6% overall, just over the 45–50% target.
  - T4 worked as intended: planned-true statements now measure true in 76% of cases.
- **Records:** median 103 words (p90 159, maximum 199), above R22's "about 70".

Targets met:
- **T1:** "request information" is top on 23.8% of the 496 next-step questions (ask ≤ 25%), in 24 of 40 families (60%).
- **T3:** 16 of 40 families (40%). When the aggravating fact is present, the main score goes to the top level in 84% of questions.
- **T4:** as above.
- **Everything else passes:**
  - format and structure;
  - `criteria: null` 35%;
  - the first-option choice share;
  - skill shares;
  - no gold zeros and no near-duplicate pairs;
  - score mass two or more levels from the top at 8%.

## Cost

| Stage | Cost (USD) |
|---|---|
| scales | 0.02 |
| family | 6.1 |
| records | 20.26 |
| judge | 2.4 |
| **total** | **28.78** |

- **By model:** {"claude-opus-5-5": 6.13, "claude-sonnet-5-5": 20.26, "claude-haiku-5-5": 2.4}.
- **Requests:** 6226 in total, 6184 of them through the Batches API.
- **Includes:** every dropped record and retry.

## Licence

Released to the Signpost-D team without restriction (CC0).
