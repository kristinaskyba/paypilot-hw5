# L05 · Multi-turn Test Plan — PayPilot

## Run
Copy `l05_dialogs.jsonl` into the L05 kit (`~/PayPilot-Kristina/l05/sets/`) and run:

    docker compose run --rm eval --set l05_dialogs --dry-run
    docker compose run --rm eval --set l05_dialogs --profile clean
    docker compose run --rm eval --set l05_dialogs --profile lesson-05

`set_hash`: bb2256591f1d (v3; v2 was 802309a7ef39, v1 was e705b4c94e56)

## Dialogue cases
| id | Role in the plan | Length · entities · switches | Invariants |
| --- | --- | --- | --- |
| CONV-H01 | main (C-07) | 7 · 5 · 1 | TX-0401 (5/7, no TX-0402), amount 240 (5/7, no 180), reason code goods_not_received (5/7, no substitute), deadline 2026-10-18 (7) |
| CONV-H02 | control | 2 · 5 · 0 | TX-0401, amount 240, reason code (2); agent.summarize must not run |
| CONV-H03 | worst case | 10 · 5 · 2 | TX-0401 (8/10, no TX-0402), amount 240 (8/10, no 180), reason code (5, and 8/10, no substitute), deadline 2026-10-18 (10) |
| CONV-H04 | axis isolation: topic switches | 7 · 5 · 0 | TX-0401 (5/7, no TX-0402), amount 240 (5/7, no 180), reason code (5/7, no substitute), deadline 2026-10-18 (7) |
| CONV-H05 | axis isolation: entities | 7 · 1 · 1 | amount 240 (5 and 7, no 180, no 480) |

*"5/7" = any of steps 5 and 7 (`probe_any`); "no X" = the substitute must not appear at those steps.*

## Set versions
- **v1** (`e705b4c94e56`, commits `00ff4a9`, prediction `566d94e`): first runs, reports in `reports/v1/`. The `clean` baseline was 2/5: the reason-code invariant required the code at the last step even when the question did not ask for it, and the deadline pattern missed "18 Oct 2026". On `lesson-05`, "240" passed even when the agent listed both 240 € charges (TX-0401 and TX-0402).
- **v2** (`802309a7ef39`): reason code on any of the probe steps, wider date patterns, new invariant TX-0401 with "no TX-0402", "no 480" in CONV-H05. Replaying the v2 checks on the v1 answers: `clean` 5/5 cases in every run.
- **v3** (`bb2256591f1d`): explicit oracle checks at the step where each value is first stated — step 1 reason code goods_not_received, steps 1–2 amount 240, step 2 deadline 2026-10-18 (H05: amount at step 1). Before v3 `oracle_turn`/`oracle_by` were only metadata. Replaying v3 on the v2 answers: oracle checks correct in all runs on both profiles.

## Runs
| Run | Profile | Result | Report |
| --- | --- | --- | --- |
| v1 baseline | `clean` | 2/5 (false fails, see "Set versions") | `reports/v1/l05_dialogs-clean-20261023T142707.json` |
| v1 lesson profile | `lesson-05` | 2/5 | `reports/v1/l05_dialogs-lesson-05-20261023T145309.json` |
| **v2 baseline** | `clean` | **5/5** (25/25 runs) | `reports/l05_dialogs-clean-20261023T151519.json` |
| **v2 lesson profile** | `lesson-05` | **1/5** | `reports/l05_dialogs-lesson-05-20261023T152706.json` |

Prediction before the `lesson-05` run:
- CONV-H01 (main): fails at step 5 — the summary at step 3 drops or substitutes the case facts (2.3: loss at 3 in 5/5 runs, visible at 5 in 4/5).
- CONV-H02 (control): passes — 2 steps, the summary never runs.
- CONV-H03 (worst case): fails at step 5 — the first return to the case comes right after the summary.
- CONV-H04 (no detour): fails at step 5 — the summary still runs at step 3; step 5 is the first check.
- CONV-H05 (one fact): passes in most runs; may fail at step 5 in 1–2 of 5 runs (240 → 180 substitution seen in the lab).
Commit with the prediction: `566d94e`
What failed (v2, `lesson-05`): CONV-H01 (0/5 runs passed, first fail at step 5 in 5/5), CONV-H03 (0/5, step 5 in 4 runs, step 8 in 1), CONV-H04 (0/5, step 5 in 4 runs, step 7 in 1), CONV-H05 (3/5 runs failed, steps 5–7: "total EUR 480.00"). CONV-H02 (control) passed 5/5.
Difference from the prediction:
- H01, H03: as predicted (fail at step 5). In v1 H01 failed only at step 7, because "240" also matched the listed TX-0402 — fixed in v2 with "no TX-0402".
- H04: predicted fail at step 5 — v1 showed 1/5 failing and looked like a surprise, but v2 shows 5/5 failing: without a detour the agent loses TX-0401 the same way (offers TX-0402 as an equal option); v1 simply did not check for it.
- H05: predicted "mostly passes, may fail at 5 in 1–2 runs" — failed in 3/5 runs. The single amount survives the summary (240 holds in 5/5), but the agent merges the two 240 € charges into "EUR 480.00", which the prediction did not foresee.
- H02: as predicted (passes).

## Extra finding (action layer)
> **Note for the mentor:** on `clean` the agent opens the dispute without an explicit "yes" when the customer only repeats the amount. Reproduced in a separate case **CONV-A01** (`l05_action.jsonl`, severity critical): **5/5 runs failed** on `clean`. Details and trace evidence: `multi-turn-test-plan.md`, section 3.4. Report: `reports/action/l05_action-clean-20261023T161432.json`.
>
>     docker compose run --rm eval --set l05_action --profile clean --traces
