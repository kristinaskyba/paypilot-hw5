# L05 · Multi-turn Test Plan — PayPilot

## Run
Copy `l05_dialogs.jsonl` into the L05 kit (`~/PayPilot-Kristina/l05/sets/`) and run:

    docker compose run --rm eval --set l05_dialogs --dry-run
    docker compose run --rm eval --set l05_dialogs --profile clean
    docker compose run --rm eval --set l05_dialogs --profile lesson-05

`set_hash`: e705b4c94e56

## Dialogue cases
| id | Role in the plan | Length · entities · switches | Invariants |
| --- | --- | --- | --- |
| CONV-H01 | main (C-07) | 7 · 5 · 1 | amount 240 (steps 5/7, no 180), reason code goods_not_received (7), deadline 2026-10-18 (7) |
| CONV-H02 | control | 2 · 5 · 0 | TX-0401, amount 240, reason code (2); agent.summarize must not run |
| CONV-H03 | worst case | 10 · 5 · 2 | reason code (5, 10), amount 240 (8/10, no 180), deadline 2026-10-18 (10) |
| CONV-H04 | axis isolation: topic switches | 7 · 5 · 0 | amount 240 (5/7, no 180), reason code (7), deadline 2026-10-18 (7) |
| CONV-H05 | axis isolation: entities | 7 · 1 · 1 | amount 240 (5 and 7, no 180) |

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
Commit with the prediction: `<hash>`
What failed: <id>. Difference from the prediction: <explanation or "none">
