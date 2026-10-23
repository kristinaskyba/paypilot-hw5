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

Prediction before the `lesson-05` run: pending
Commit with the prediction: `<hash>`
What failed: <id>. Difference from the prediction: <explanation or "none">
