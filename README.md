# L05 · Multi-turn Test Plan — PayPilot

## Run
Copy `l05_dialogs.jsonl` into the L05 kit (`~/PayPilot-Kristina/l05/sets/`) and run:

    docker compose run --rm eval --set l05_dialogs --dry-run
    docker compose run --rm eval --set l05_dialogs --profile clean
    docker compose run --rm eval --set l05_dialogs --profile lesson-05

`set_hash`: 802309a7ef39 (v2; v1 was e705b4c94e56)

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

## Runs
| Run | Profile | Result | Report |
| --- | --- | --- | --- |
| Baseline | `clean` | <N/M> | `reports/…` |
| Lesson profile | `lesson-05` | <N/M> | `reports/…` |

Prediction before the `lesson-05` run:
- CONV-H01 (main): fails at step 5 — the summary at step 3 drops or substitutes the case facts (2.3: loss at 3 in 5/5 runs, visible at 5 in 4/5).
- CONV-H02 (control): passes — 2 steps, the summary never runs.
- CONV-H03 (worst case): fails at step 5 — the first return to the case comes right after the summary.
- CONV-H04 (no detour): fails at step 5 — the summary still runs at step 3; step 5 is the first check.
- CONV-H05 (one fact): passes in most runs; may fail at step 5 in 1–2 of 5 runs (240 → 180 substitution seen in the lab).
Commit with the prediction: `566d94e`
What failed: <id>. Difference from the prediction: <explanation or "none">
