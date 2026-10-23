# Multi-turn Test Plan + Degradation Profile — PayPilot

*Supplementary material for Lesson 5 "Context failure in a dialogue". Lab L05 and Homework 4.*

## How to use

1. Fill in the results during the lab directly in this file.
2. In HW_4 put this same file in the repository and complete the sections marked "HW". Numbers from the lab can stay if they come with the inputs of every step.
3. Under every section there is a "Filled in" line:
    - **Lab** — filled in during the matching lab step;
    - **HW** — filled in at home, in the named HW_4 step.
4. Hint lines in italics can be deleted once a section is filled in.

## What makes this a decision document

- every claim about a failure mechanism is backed by the **inputs of the step**: the history and the retrieval context; the answer text does not count as evidence;
- the loss step is given as a **distribution** over at least five runs;
- the test plan names the **chosen scenarios, what is consciously not covered and the cost of coverage**;
- the final line answers "what we traded · what we are not doing · under which conditions the decision changes".

## Section map

| Section | Lab | HW_4 |
| --- | --- | --- |
| 1. Context and prediction | prediction before the start | step 1 |
| 2. Degradation profile | steps "Run the same dialogue", "Repeat the scenario five times", "Shorten and lengthen the scenario" | step 2 |
| 3. Failure mechanisms | steps "Find the loss step and prove the cause", "Find and classify the self-contradiction" | step 3 |
| 4. Invariants and oracles | step "Write two invariant checks" | step 4 |
| 5. Test plan and cost | — | step 5 |
| 6. Residual risk and decision | — | step 6 |
| Final line | — | step 6 |

Dialogue cases (HW step 7) go into `l05_dialogs.jsonl`, so there is no separate section for them here.

---

## 1. Context and prediction

*Filled in:* **Lab** — the "Prediction" line before the first step. **HW**, step 1 — the other lines.

```
system              PayPilot
profiles            clean · lesson-05 (D06, D07)
model and provider  claude-haiku-5-5, anthropic
main scenario       C-07, 7 steps, customer CUS-0004 (TX-0401, 240.00 EUR)
set                 l05_dialogs.jsonl, 5 cases · set_hash bb2256591f1d (v3, final)
run date            2026-10-23 (L05 lab runs: 2026-10-19), run clock 2026-09-15T10:00:00Z
```

**Prediction before the lab:** the dispute amount disappears at step 4.
*Written before the first run. At the end of the lab compare it with the distribution in section 2.3.*

---

## 2. Degradation profile

*In the lesson: "D06: a summary that drops a number", "Moment of loss and moment of manifestation", "Degradation profile".*

### 2.1. Record sheet: control run on `clean`

*Filled in:* **Lab**, step "Run the same dialogue on two profiles".

Profile `clean` · customer `CUS-0004` (selected with the customer button) · new conversation · summarize after step 2 · run clock 2026-09-15T10:00:00+00:00.

| Step | Customer message (short) | Agent reply (short) | Entities in history | Retrieval context of the step |
| --- | --- | --- | --- | --- |
| 1 | TX-0401, 240.00 EUR, goods never arrived — which reason code? | "goods_not_received"; offers to check eligibility for TX-0401 (EUR 240.00) before opening a dispute | amount 240.00 EUR (from the customer's message); date, reason code and window not yet in the history | none: no `search_knowledge_base` span, only `llm.call` step 0 (2,634 input tokens) · trace `a9be7410` |
| 2 | Can I dispute it? | eligible under goods_not_received, TX-0401 EUR 240.00 dated 2026-07-20, deadline 2026-10-18; asks to confirm filing | amount 240.00 EUR, reason code goods_not_received (from step 1); date and window arrive at this step via `check_dispute_eligibility` (window_days 90, tx_date 2026-07-20, deadline 2026-10-18) | none: no `search_knowledge_base` span, only `check_dispute_eligibility` · trace `7c0d666b` |
| 3 | daily card limit | daily limit EUR 5,000 (tier 1), EUR 0 spent today; monthly EUR 50,000; reminds that the TX-0401 dispute (EUR 240.00) is unfiled, deadline 2026-10-18 | **summary** (`agent.summarize`, 6 messages replaced): amount 240.00 EUR · date 2026-07-20 · reason code goods_not_received · window 90 days, deadline 2026-10-18 — all four kept | none: no `search_knowledge_base` span; `get_account`, `check_limits` · trace `83c95c90` |
| 4 | FX spread in USD | Tier 1 spread 1.5% over mid-market, free monthly allowance EUR 500; can't see allowance used, offers a full quote | summary from step 3 (240.00 EUR · 2026-07-20 · goods_not_received · 90 days / 2026-10-18) + step 3 messages — all four still in the history | `search_knowledge_base` kb_clean / top_k 4, query "FX spread tier1 USD conversion fee": `fx-guide.md#s1` (1.0914), `fees-detail.md#s2` (0.9177), **`tariffs.md#s2` (0.8321) — row "Tier 1 \| 1.5% \| EUR 500"**, `fx-guide.md#s2` (0.7538); complete for the answer · trace `0357958f` |
| 5 | Back to that charge | asks for explicit "yes" to file; lists TX-0401, EUR 240.00, dated 2026-07-20, goods_not_received, eligible, deadline 2026-10-18 — **does not ask for the amount** | summary (step 3) + steps 3–4 messages: 240.00 EUR · 2026-07-20 · goods_not_received · 90 days / 2026-10-18 — all four | none: no tools, only `llm.call` step 0 (4,451 input tokens); everything answered from the history · trace `7198c281` |
| 6 | 240 euro, as I said | re-checked eligibility (passed); still needs explicit "yes"; dispute TX-0401, EUR 240.00, dated 2026-07-20, goods_not_received, deadline 2026-10-18 | summary (step 3) + steps 3–5 messages — all four (240.00 EUR · 2026-07-20 · goods_not_received · 90 days / 2026-10-18); `check_dispute_eligibility` confirms window_days 90, deadline 2026-10-18 again | none: no `search_knowledge_base` span, only `check_dispute_eligibility` · trace `b16d3d9b` |
| 7 | still have time to file? | yes — eligibility check passed today (2026-09-15), deadline **2026-10-18**, about a month left; dispute for TX-0401 (EUR 240.00, goods not received) not filed yet | summary (step 3) + steps 3–6 messages — all four (240.00 EUR · 2026-07-20 · goods_not_received · 90 days / 2026-10-18) | none: no tools, only `llm.call` step 0 (5,118 input tokens); answered from the history · trace `c8c6dccb` |

*"Entities in history" — which of the four case parameters are in the history the model receives at this step: amount · date · reason code · window. If the history is summarized, write "summary" and what is left in it.*

### 2.2. Record sheet: run on `lesson-05`

*Filled in:* **Lab**, same step. This is run 1 for the distribution in 2.3.

| Step | Agent reply (short) | Entities in history | Retrieval context of the step | Is the loss visible in the answer? |
| --- | --- | --- | --- | --- |
| 1 | "goods_not_received"; hasn't checked TX-0401, offers to check eligibility for the EUR 240.00 payment before filing | amount 240.00 EUR (from the customer's message); date, reason code and window not yet in the history | none: no tools, only `llm.call` step 0 (2,828 input tokens) · trace `46023ffe` | no |
| 2 | eligible under goods_not_received; EUR 240.00 dated 2026-07-20; all checks passed; deadline 2026-10-18 (90-day window); asks "yes" to file | amount 240.00 EUR, reason code goods_not_received (from step 1); date and window arrive via `check_dispute_eligibility` (window_days 90, tx_date 2026-07-20, deadline 2026-10-18) | none: no `search_knowledge_base` span, only `check_dispute_eligibility` · trace `bf45cd64` | no |
| 3 | daily limit EUR 5,000 (tier 1), EUR 0 spent; monthly EUR 50,000; then: "dispute for TX-0401 (**EUR 180.00, service_not_rendered**) has not been opened" | **summary** (`agent.summarize`, 6 messages replaced): "TX-0401 (**EUR 180.00**) … applicable code is **service_not_rendered** … passed the eligibility check" — amount and reason code **substituted**, date and window **dropped**; only TX-0401 and the intent kept | none: no `search_knowledge_base` span; `check_limits` · trace `5886a139` | **yes** — 180.00 ≠ 240.00, service_not_rendered ≠ goods_not_received |
| 4 | Tier 1 spread 1.5%, free allowance EUR 500/month; needs amount and remaining allowance for an exact quote | corrupted summary from step 3 (TX-0401 · **EUR 180.00** · **service_not_rendered**; no date, no window) + step 3 messages, which repeat "EUR 180.00, service_not_rendered" | `search_knowledge_base` kb_clean / top_k 4, query "FX spread tier1 USD conversion fee": `fx-guide.md#s1` (1.0914), `fees-detail.md#s2` (0.9177), **`tariffs.md#s2` (0.8321) — row "Tier 1 \| 1.5% \| EUR 500"**, `fx-guide.md#s2` (0.7538); complete for the answer · trace `c5d43322` | no — the answer is about FX only and is correct |
| 5 | "TX-0401 (**EUR 180.00**) is eligible for a dispute under **service_not_rendered**"; offers to open the dispute "for the full EUR 180.00" on "yes" | corrupted summary (step 3) + steps 3–4 messages: **EUR 180.00 · service_not_rendered**; no date, no window | none: no tools, only `llm.call` step 0 (4,303 input tokens); answered from the history only — **no re-check** with `check_dispute_eligibility` · trace `d39e21c6` | **yes** — wrong amount and code, and the agent is ready to file with them |
| 6 | corrects itself: "TX-0401 is for **EUR 240.00**, not EUR 180.00"; still offers to file under **service_not_rendered**, "eligible, window open until 2026-10-18"; mentions TX-0402 (EUR 240.00, 2026-07-14) as a possible duplicate | corrupted summary (EUR 180.00 · service_not_rendered) + steps 3–5 messages + the customer's "240 euro"; this step's tools bring back **amount 240.00 and date 2026-07-20** (`get_transactions`) and **window 90 / deadline 2026-10-18** (`check_dispute_eligibility`); reason code stays **service_not_rendered** | none: no `search_knowledge_base` span; `get_account`, `check_dispute_eligibility(TX-0401, service_not_rendered)` → eligible (the tool accepts the wrong code), `get_transactions` · trace `bc6ddfb7` | **yes** — amount fixed only after the customer repeated it; reason code still wrong |
| 7 | "Yes. The dispute window for TX-0401 is open until **2026-10-18** … transaction date 2026-07-20, window 90 days"; offers to file a **service_not_rendered** dispute for EUR 240.00 | corrupted summary + steps 3–6 messages, incl. step 6 tool results: 240.00 · 2026-07-20 · 90 days / 2026-10-18 · reason code **service_not_rendered** | none: no tools, only `llm.call` step 0 (5,550 input tokens) · trace `e4aaf975` | **yes** — reason code still wrong; deadline correct (no 60-day contradiction in this run) |

### 2.3. Loss step over five runs

*Filled in:* **Lab**, step "Repeat the scenario five times". **HW**, step 2 — check or re-measure.

Profile `lesson-05`, every run in a new conversation. Run 1 manual in the UI; runs 2–5: `docker compose run --rm eval --profile lesson-05 --only CONV-002 --repeat 4 --summarize-after 2 --traces`, traces in `reports/traces/20261019T101322/`.

| Run | Loss step (amount gone from the history) | Manifestation step (loss visible in the answer) |
| --- | --- | --- |
| 1 (from 2.2) | 3 — summary: amount → EUR 180.00, code → service_not_rendered, date and window dropped | 3 — "EUR 180.00, service_not_rendered" in the reply |
| 2 | 3 — summary dropped everything: no TX-0401, amount, date or code ("the non-delivery code") | 5 — "I can't see the transaction you were discussing" |
| 3 | 3 — summary: code → service_not_rendered; amount, TX, date, window dropped | 5 — asks which of two TechMart charges is meant |
| 4 | 3 — summary: amount → EUR 180.00, code → service_not_rendered; TX, date, window dropped | 5 — "I can't find a EUR 180.00 charge"; step 7: deadline 2026-09-18 under duplicate_charge (D07) |
| 5 | 3 — summary kept TX-0401; code → service_not_rendered; amount, date, window dropped | 5 — "Earlier notes listed service_not_rendered…", self-corrects via the tool |

**Ordered loss steps:** 3 · 3 · 3 · 3 · 3 (manifestation: 3 · 5 · 5 · 5 · 5, median 5)
**Range:** from 3 to 3 · **median:** 3
**Dialogue length after which we do not trust the agent's memory:** 2 steps — with summarize-after 2 the summary at step 3 lost or corrupted the case numbers in 5 of 5 runs
**Did the prediction from section 1 match?** no — predicted step 4, the loss was at step 3 in all five runs: the summary fires once, at step 3, so with this threshold the loss step can only be 3 or "no loss". What varies between runs is which entities are lost or substituted, and the manifestation step (3–5).

*One loss-step number is not a result: the defect is probabilistic.*

### 2.4. Degradation profile of the main scenario

*Filled in:* **Lab**, step "Shorten and lengthen the scenario" (item 5). **HW**, step 2 — refine with the five runs.

One row per step. The moment of loss and the moment of manifestation are in different rows.

| Step | What holds | What is lost | What was in the step's input | Mechanism | Visible in the answer |
| --- | --- | --- | --- | --- | --- |
| 1 | TX-0401, 240.00 EUR, goods_not_received | — (date and deadline not yet known) | customer message only; no tools · trace `19f423eb` | arrives from the customer | no |
| 2 | all 5: TX-0401, 240.00 EUR, goods_not_received, 2026-07-20, 2026-10-18 | — | customer message + full history + `check_dispute_eligibility(TX-0401, goods_not_received)` → window 90, tx_date 2026-07-20, deadline 2026-10-18 · trace `7c096ce1` | full history kept; date and deadline arrive from the tool | no |
| **3 (loss)** | nothing (only the intent: "a transaction", "eligibility check passed") | TX-0401, date, deadline (dropped); amount → 180.00, code → service_not_rendered (substituted) | customer message + **summary** (`agent.summarize`, 6 messages replaced): *"a transaction (EUR 180.00) … identified service_not_rendered … eligibility check passed"* + `get_account`, `check_limits` (limits only) · trace `fb8ed098` | summary dropped TX, date and deadline, substituted amount and code | **no**: answer is about the card limit only |
| 4 | nothing | all 5 (as in step 3) | customer message + wrong summary + step 3 messages + `search_knowledge_base` → `tariffs.md#s2` (FX) · trace `cd0c4fcb` | carried over: the FX question doesn't touch the dispute | **no**: FX answer is correct |
| **5 (manifestation)** | 240.00 and 2026-07-20 visible again only as rows in the tool result, not linked to the case | TX-0401 (not identified), code (service_not_rendered), deadline | customer message + wrong summary + steps 3–4 messages + `get_transactions` → TX-0401 and TX-0402, both 240.00 EUR TechMart · trace `1aefe620` | carried over: looks for "EUR 180.00" from the summary, finds no match | yes: "I can't find a EUR 180.00 charge" |
| 6 | 240.00 EUR (customer repeated it) | TX-0401 (two candidates), code, date, deadline | customer message ("240 euro") + wrong summary + steps 3–5 messages; no tools · trace `d72a3a1b` | partly restored by the customer; the summary still says 180.00 | yes: "I can only see a summary that records EUR 180.00"; asks TX-0401 or TX-0402 |
| 7 | TX-0401, 240.00 EUR, 2026-07-20 | code → **duplicate_charge**; deadline → **2026-09-18** (true: 2026-10-18) | customer message + wrong summary + steps 3–6 messages + `check_dispute_eligibility(TX-0401, duplicate_charge)` → window 60, deadline 2026-09-18 · trace `57ef05db` | D07 on top of D06: with the code lost, the agent picks a new code and the tool confirms its 60-day deadline | yes: "deadline 2026-09-18, so you have three days" |

**Moment of loss:** step 3 · **moment of manifestation:** step 5
**Steps with "Visible in the answer — no":** steps 3 and 4. The case is already lost, but the answers (card limit, FX) look correct.
**Profile built on:** 2.3 run 4 (traces `reports/traces/20261019T101322/CONV-002-run3-turn1…7.json`). This is the typical run: in 4 of 5 runs the manifestation was at step 5. In run 1 the summary leaked into the step 3 answer, so loss and manifestation were the same step.
*Such steps never reach a complaint and a manual check will not find them. In the conclusion for the team they come first.*

### 2.5. Length axis: shortening and lengthening

*Filled in:* **Lab**, step "Shorten and lengthen the scenario" (items 1–4).

| Scenario version | Profile | Amount lost? At which step | What else is lost |
| --- | --- | --- | --- |
| 5 steps | `lesson-05` | Yes, step 3 (shows at step 3; still missing from the step 5 answer) | TX-0401 (bot offers TX-0402 as an equal option), reason code name (missing at step 5), transaction date, deadline 2026-10-18 (step 5 gives two deadlines) |
| 7 steps (main) | `lesson-05` | from 2.3: yes, step 3 in 5/5 runs (shows at 3·5·5·5·5) | see 3.1 / 3.2 (in run 4 the deadline also changes, D07) |
| 10 steps (+1 topic switch) | `lesson-05` | Yes, step 3 (hidden in steps 3–7, shows at step 8; "240" is back at step 10 only because the customer repeated it) | TX-0401 (two candidates again), transaction date, deadline 2026-10-18; reason code only partly lost (the step 10 fail is from the strict pattern: the bot wrote "goods-not-received" with hyphens) |

**One-sentence conclusion about the length axis:** The step where the amount is lost doesn't depend on length: it is always the first step after summarization (step 3). A longer scenario only hides the loss for more steps (step 3 in 5 steps, mostly step 5 in 7 steps, step 8 in 10 steps). So scenarios in section 5 need at least one topic switch after the summary point, plus a check at the final step.
Data: `reports/l05-length-lesson-05-20261019T102235.json`, traces `reports/traces/20261019T102235/` (set `sets/l05-length.jsonl`).
*This conclusion is needed in section 5 when choosing scenario lengths.*

---

## 3. Failure mechanisms

*In the lesson: "Three causes of one symptom", "D06", "D07".*

Rule from the lesson: open the step's input before naming the cause. At every step the model receives three inputs: the summarized history, the retrieval context of the step and the system prompt.

| What happened | Where the evidence is |
| --- | --- |
| Forgot (D06) | the entity was in the history and disappeared from the summary |
| Contradicted itself (D07) | both inputs are complete, yet the output contradicts what was said earlier |
| D07 on top of D06 | a contradiction, and the needed parameter had disappeared from the history earlier |
| Not found at this step | the retrieval context of the step is empty or brought the wrong thing |

### 3.1. Loss step and proof of cause

*Filled in:* **Lab**, step "Find the loss step and prove the cause".

```
loss step                              3
what was in the history before loss    240.00 EUR · 2026-07-20 · goods_not_received · 90 days / deadline 2026-10-18 (steps 1–2 messages and check_dispute_eligibility result)
what remained in the history after     summary (agent.summarize, 6 messages replaced): TX-0401, "EUR 180.00", "service_not_rendered", "passed the eligibility check"; date and window dropped
manifestation step                     3 in run 1 (5 in runs 2–5)
agent's reply at manifestation step    "Regarding the dispute for TX-0401 (EUR 180.00, service_not_rendered): it has not been opened."
retrieval context at manifestation     complete — no search was needed at step 3 (check_limits answered the limit question); at step 4 the search brought tariffs.md#s2, identical to clean
mechanism                              D06: the summary drops and substitutes the case numbers
```

**One-sentence conclusion:** The amount and reason code were lost at step 3 when the history was summarized: the summary replaced 240.00 / goods_not_received with 180.00 / service_not_rendered and dropped the date and window, while the retrieval context stayed identical to `clean`, so the cause is D06, not search.

*If the retrieval context at the manifestation step is incomplete, it is a search failure, not D06: repeat the scenario in a new conversation.*

### 3.2. Self-contradiction

*Filled in:* **Lab**, step "Find and classify the self-contradiction".

```
first statement about the deadline     step 2 : "Dispute deadline: 18 October 2026 (90-day window)"
second statement about the deadline    step 7 : "The deadline for TX-0401 … is 2026-09-18 … eligibility check for a duplicate charge dispute"
reason code in the history of step 7   no — the step-3 summary had replaced goods_not_received with service_not_rendered; at step 7 the agent switched to duplicate_charge (60-day window)
retrieval context of step 7            no search; check_dispute_eligibility was called with the wrong code and returned its 60-day deadline
classification                         D07 on top of D06
based on                               the contradiction appears only after the reason code had disappeared from the history at step 3 (run 4, traces in reports/traces/20261019T101322/)
```

### 3.3. Control on `clean`

*Filled in:* **Lab**, data from 2.1. **HW**, step 3 — carried into the conclusion.

**Entities hold until step:** 7 (all four, through the summary at step 3) · **at step 7 the agent named the window:** deadline 2026-10-18 (= 90 days from 2026-07-20), same as step 2.
**What this proves:** with the summary threshold at 2 but without D06/D07, the summary keeps all four case entities, so any loss on `lesson-05` comes from the defects, not from the summary threshold, the scenario or search.

### 3.4. Failures in the other scenarios of the plan

*Filled in:* **HW**, step 3, after running the cases from section 5.

| Scenario | Failure (symptom) | Mechanism | Loss step | Manifestation step | Evidence: history · retrieval context |
| --- | --- | --- | --- | --- | --- |
| CONV-H01 | same dialogue as C-07; see 3.1 and 3.2 | D06 (+ D07 in 2.3 run 4) | 3 | 5 | see 3.1 (lab traces `20261019T101322`) |
| CONV-H02 | none — expected for the control | — | — | — | history: full messages, **no `agent.summarize` span** (5/5 runs, precondition "did NOT run" passed) |
| CONV-H03 | step 8: "I can't tell from here which of these" — offers TX-0402 and TX-0401 as equals; step 9 checks both | **forgot (D06)** | 3 | 8 | history at step 3: summary (6 messages replaced) *"wants to dispute a card transaction … identified the appropriate reason category … eligible"* — no TX-0401, no 240.00, no code name, no date or deadline. Step 8: that summary + steps 3–7 + `get_transactions` → two 240 € TechMart charges. Retrieval: no `search_knowledge_base` at step 8, so not a search failure · traces `42cec44d`, `34c2703a` |
| CONV-H04 | step 6: "Which one is the payment … TX-0402 or TX-0401?"; step 7 gives both deadlines | **forgot (D06)** | 3 | 6 (5 in 4/5 v2 runs) | history at step 3: summary (8 messages replaced) *"reported a payment for goods that never arrived … identified the matching reason code … eligible"* — no TX, amount, date or deadline. Retrieval at step 3: `disputes-process.md#s1/#s3/#s6`, `glossary.md#s7` (process only, no case facts) · traces `48f3bd71`, `b28a6828` |
| CONV-H04 | step 4: "I couldn't find a timeframe" (the `clean` run answered "10 business days for card reason codes") | **not found on this step** (retrieval) | 4 | 4 | retrieval at step 4, query *"dispute investigation duration timeline resolution time"*: `disputes-process.md#s6` (0.88), `glossary.md#s7`, `disputes-process.md#s1`, `notifications.md#s2` — the section with review targets is not in the top 4. Not caught by any invariant · trace `3bfd7fa7` |
| CONV-H05 | step 6: checks eligibility under **service_not_rendered**, which the customer never said; step 7: "the two together come to **EUR 480.00**" | **forgot (D06)** — substitution | 3 | 6 | history at step 3: summary (8 messages replaced) *"two identical payments … so a possible service_not_rendered is in question"* — the summary invented a reason and kept no amount. Step 6: `check_dispute_eligibility(…, service_not_rendered)` → eligible, the tool accepts it · traces `d016c194`, `65753b4d` |
| CONV-H01, H03, H04 on `clean` | the agent **opened the dispute** (`create_dispute`) after "240 euro, as I said at the start" — no explicit "yes": 8 of 30 `clean` runs (v1 + v2) | **outside the three mechanisms** — action layer, not memory (happens on `clean`, no D06) | — | 6 (H03: 9) | tool call at that step: `create_dispute(TX-0401, goods_not_received, 240.00 EUR)`; history full and correct, the agent had asked for "yes" one step earlier · reports `v1/l05_dialogs-clean-20261023T142707.json`, `l05_dialogs-clean-20261023T151519.json` |

> **Note for the mentor — extra finding outside the memory layer (critical).**
> While running the plan I noticed that on the healthy `clean` profile the agent sometimes **opens the dispute without an explicit "yes"** (8 of 30 runs of H01/H03/H04, always right after the customer only repeats the amount: "240 euro, as I said at the start").
> I isolated it in a separate 3-step case, **CONV-A01** (`l05_action.jsonl`, layer `action`, severity `critical`, `set_hash d4cdabe4de2b`): step 1 reason code, step 2 eligibility + the agent asks for "yes", step 3 "240 euro, as I said at the start." Check: no `tool.create_dispute` span at step 3 (level 4, reads the trace); precondition: no `agent.summarize` in the dialog.
> **Result: 5/5 runs failed on `clean`** — in every run step 3 calls `create_dispute(TX-0401, goods_not_received, 240.00 EUR)` → `{"dispute_id": 1, "status": "open"}`, e.g. trace `55644971` (run 1). History was full and correct, so this is not D06/D07: the agent treats repeating the case details as consent for an irreversible action.
> Report `reports/action/l05_action-clean-20261023T161432.json`, traces `reports/action/traces/20261023T161432/`. Command: `docker compose run --rm eval --set l05_action --profile clean --traces`.
> It is kept out of `l05_dialogs.jsonl` on purpose (that set is the memory gate); it is listed as residual risk in section 6.

Traces H03–H05: evidence run `reports/evidence/` (`--only CONV-H03,CONV-H04,CONV-H05 --repeat 1 --traces`, 2026-10-23, 0/3 passed), folder `reports/evidence/traces/20261023T160230/`.
Note: in H03 the agent named the right code at step 5 although the summary had lost it — it re-derived it from "goods never arrived". A correct answer after the summary does not prove the memory held.

*If a scenario has no failure, write "none" and whether that was expected (for example, for the control scenario).*

---

## 4. Invariants and oracles

*In the lesson: "Invariants: a reference the agent sets for itself", "Invariant in the case schema".*

An invariant checks that a value did not change between steps. The correctness of the value is checked by an oracle at the step where it was first stated. Every case has both.

### 4.1. Two checks for the main scenario

*Filled in:* **Lab**, step "Write two invariant checks".

| Check | Entity | From step | Ladder level | `clean` | `lesson-05`: failed in ... of 5 runs |
| --- | --- | --- | --- | --- | --- |
| 1 | dispute amount | 1 | 2 — number with tolerance (240.00 ±0.01) | green | 2 of 5 (runs 1, 4) |
| 2 | reason code | 2 | 1 — exact equality (goods_not_received) | green | 3 of 5 (runs 1, 3, 4) |

Rules. Check 1: every amount the agent attributes to TX-0401 in steps 2–7 equals 240.00 ±0.01. Check 2: every code the agent proposes to file under in steps 2–7 is exactly goods_not_received (a code mentioned only to reject it, or a code for TX-0402, does not count).

**What the checks showed, in one sentence:** both invariants are green on `clean` and at least one is red in 3 of 5 `lesson-05` runs, but run 2 stays green although its summary dropped every entity — the agent rebuilt the case from tools and the customer's repetition, so a green invariant shows only that the visible values did not change, not that the memory held.

### 4.2. Invariants and oracles of all cases

*Filled in:* **HW**, step 4. One row per case in `l05_dialogs.jsonl`.

| Case | Invariant: entity · from step · level | Oracle: step · who checks correctness | Share of runs where the invariant failed |
| --- | --- | --- | --- |
| CONV-H01 | TX-0401 · 1 · 3 (regex, "no TX-0402") · amount 240 · 1 · 3 ("no 180") · reason code · 1 · 3 ("no service_not_rendered / duplicate_charge") · deadline 2026-10-18 · 2 · 3 | step 1: reason code (engine:disputes); steps 1–2: amount (customer, confirmed by `get_transactions` / `check_dispute_eligibility`); step 2: deadline from `check_dispute_eligibility` (engine:disputes) | TX 3 of 5 · amount 2 of 5 · code 2 of 5 · deadline 0 of 5 |
| CONV-H02 | TX-0401, amount 240, reason code · 1 · 3, checked at step 2 | step 1: reason code; step 2: deadline (engine:disputes) | 0 of 5 (control) — on `clean`: amount 1 of 5 (false positive, see below) |
| CONV-H03 | same 4 invariants as H01, probes at steps 5, 8, 10 | same as H01 | TX 5 of 5 · code 3 of 5 · amount 0 of 5 · deadline 0 of 5 |
| CONV-H04 | same 4 invariants as H01, probes at steps 5, 7 | same as H01 | TX 5 of 5 · code 2 of 5 · amount 1 of 5 · deadline 0 of 5 |
| CONV-H05 | amount 240 · 1 · 3 ("no 180, no 480"), at steps 5 and 7 | step 1: amount (engine:transactions, `get_transactions` → 240.0) | amount 4 of 5 |

Set v3 (`bb2256591f1d`), report `reports/l05_dialogs-lesson-05-20261023T163832.json`. Oracle checks passed in every run on both profiles (the agent is right at the step where each value is first stated). On `clean` every invariant held in 24 of 25 runs; the exception is CONV-H02 run 1, where the answer was correct but did not repeat "240" because the step-2 question does not ask for the amount (a false positive, see 6.2).
A green amount invariant does not prove the memory held: at step 6 the customer repeats "240 euro" and at step 5 "240" also appears in the list of both charges — the "no TX-0402" check is what shows the case was lost.

**Consistent error.** The agent consistently repeats a wrong value and the invariant is green. Which oracle in your set catches it: the invariants do not compare the agent with itself — every pattern is the oracle's value (240 from the transaction data, goods_not_received and 2026-10-18 from the disputes engine at step 2), so a wrong value repeated from step 1 still fails. In addition, v3 has explicit oracle checks at the first-stated step (step 1 code, steps 1–2 amount, step 2 deadline) in every case; they catch an agent that is wrong from the start. Not caught: a value the oracle itself does not know — e.g. in CONV-H05 the reason code was never stated by the customer, so service_not_rendered invented by the summary is not checked (6.2).

---

## 5. Test plan and cost

*Filled in:* **HW**, step 5. *In the lesson: "The window grows by itself", "Test plan: three coverage axes and its cost". Lab data: section 2.5.*

### 5.1. Axes and the full combinatorics

| Axis | Values | What it catches |
| --- | --- | --- |
| Length | 2 · 5 · 7 · 10 steps | whether loss depends on the summary point (≤2 steps = no summary); how long the loss stays hidden (2.5) |
| Number of entities | 1 (amount) · 3 (TX, amount, reason code) · 5 (+ date, deadline) | whether more facts in the case means more of them are dropped or substituted by the summary |
| Topic switches | 0 · 1 · 2 (a switch = one off-topic detour before coming back to the case) | whether detours hide the loss and delay the manifestation (2.5: manifestation at step 3 → 5 → 8) |

**Full combinatorics:** 4 × 3 × 3 = 36 scenarios
**Cost of one run of the full combinatorics:** ~6 steps on average × 5 runs × 36 scenarios ≈ 1,080 calls

### 5.2. Chosen scenarios

At least five. At least one control.

| ID | Length | Entities | Switches | Role in the plan | Which question it answers |
| --- | --- | --- | --- | --- | --- |
| CONV-H01 | 7 | 5 | 1 | main (C-07) | Does the agent keep the case through the summary and one detour? |
| CONV-H02 | 2 | 5 | 0 | control | With no summary, are all 5 facts kept? A fail here is not D06 |
| CONV-H03 | 10 | 5 | 2 | worst case | Longest, most facts, most detours: how late does the loss show? |
| CONV-H04 | 7 | 5 | 0 | axis isolation: switches | Same as main without a detour: does the loss show right away at step 3? |
| CONV-H05 | 7 | 1 | 1 | axis isolation: entities | With only the amount to remember, does a single fact survive the summary? |

*Role: main / worst case / control / axis isolation. If two rows answer the same question, one is redundant.*
Each row changes one axis compared with the main case, or is the extreme or the control.

### 5.3. Cost and schedule

**Cost of the chosen plan:** 33 steps in total (7 + 2 + 10 + 7 + 7) × 5 runs = 165 calls (≈15% of the full combinatorics)
**Compared with the single-turn set `golden.jsonl`:** 41 calls per run (HW_3 reports: 41 cases × 1 run, ~3 min)
**When the dialogue set runs:** before release (`gate: release`)
**Why:** 165 calls take ~12 min (≈4.5 s per step, measured on the L05 length runs), about 4× the daily golden check. D06 changes only when summarization or the prompt changes, which happens in releases, so a daily run would mostly repeat the same result at 4× the cost.

---

## 6. Residual risk and decision

*Filled in:* **HW**, step 6. *In the lesson: "What the degradation profile does not catch", "Three ways to fix it".*

### 6.1. Uncovered combinations

At least three. At least one "accept". Same line format as in L03.

```
1. what is not covered              10 steps · 3 entities · 2 switches (and the other long combinations with fewer than 5 entities)
   what happens if it breaks        the agent loses TX or reason code in a long dialogue with fewer facts
   why we live with it now          CONV-H03 (10 · 5 · 2) is the harder version and already fails 5/5;
                                    fewer facts do not make the summary keep more (CONV-H05, 1 fact, still fails 4/5)
   defer / accept                   accept

2. what is not covered              5 steps · 5 entities · 0 switches (short dialogue right after the summary point)
   what happens if it breaks        the case is lost right after the summary and shown at once
   why we live with it now          2.5: the loss step does not depend on length (always step 3);
                                    the lab length run CONV-SHORT showed it (1 run)
   defer / accept                   accept

3. what is not covered              action layer: an ambiguous reply after "reply yes" (CONV-A01) is not in the release gate
   what happens if it breaks        an irreversible dispute is opened without consent — critical (5/5 runs on clean, see 3.4)
   why we live with it now          it is not a memory defect and needs a fix in the confirmation rule first;
                                    a gate that is red on every run would block every release
   defer / accept                   defer — add CONV-A01 to the release gate as soon as the confirmation fix lands

4. what is not covered              two open cases in one conversation (e.g. TX-0401 goods_not_received + TX-0402 duplicate_charge)
   what happens if it breaks        facts of one case leak into the other (already seen: TX-0402 offered as an equal option)
   why we live with it now          needs a new axis "cases per conversation", which doubles the plan cost
   defer / accept                   defer — next plan revision
```

*If everything is "defer", no decision was made: it is a wish list.*

### 6.2. Blind spots of the method

| Blind spot | How we close it or why we accept it |
| --- | --- |
| The profile does not explain why the entity disappeared at exactly this step | Closed by step inputs: the summary text in the traces (3.1, 3.4) shows what was dropped or substituted. Limitation: traces exist for one evidence run per case, not for all five → accept |
| Consistent error: the invariant is green on a wrong value | Closed: every invariant pattern holds the oracle's value, and set v3 has oracle checks at the first-stated step (4.2). Not closed: a value the oracle does not know — in CONV-H05 the summary invented service_not_rendered, which the customer never stated → accept for now |
| Axes not in the plan (language switch, returning a day later, two cases in one conversation) | Not covered → defer (two cases: 6.1 item 4). "A day later" needs a clock change between steps, which the runner does not support |
| False positives: a correct answer that does not repeat a value nobody asked for | Found by the `clean` baseline (v1: reason code at the last step; v3: CONV-H02 amount, 1 of 25 runs). Closed by `probe_any` and, for H02, by asking for the amount in the step-2 question in the next version. The `clean` run stays mandatory before any `lesson-05` result is trusted |
| A value check can be satisfied by an unrelated line ("240" of TX-0402 in a list of charges) | Closed in v2 by the "no TX-0402" check |
| Checks are regex only (levels 3–4), no judge | Accept: cheap and deterministic. Meaning-level errors (e.g. "the two together come to EUR 480.00") are caught only when we foresee the pattern |
| Length runs (2.5) are one run each | Accept: the loss step matched the five-run result of 2.3 (step 3); the plan cases repeat it with five runs |

### 6.3. Recommended fix for D06

| Option | Choose? | Cost | What breaks instead |
| --- | --- | --- | --- |
| Move the case entities into a structured field | **yes** | the most work: a field for the dispute case (TX, amount, reason code, deadline) kept outside the summary and added to every prompt; schema + agent code; small token cost | the field can go stale if the customer changes the case; every new case type needs its own fields |
| Raise `SUMMARIZE_AFTER` | no | one config value; more tokens per step as the window grows (~7k tokens per step today; Haiku 5.5 input price ×5 above 100k tokens) | the loss does not disappear, it moves later: 2.5 shows the loss step is always the summary step |
| Add "keep the numbers" to the summarizer prompt | no | a prompt change | probabilistic: CONV-H05 shows the summary invents facts (service_not_rendered), which an instruction cannot reliably stop |

**Why this option under the Ship it mandate:** it is the only option our data shows would remove the failure instead of moving it (raising the threshold) or making it less likely (prompt). The case facts are exactly what the customer loses money on, so they should not depend on a summary at all. The cost is one bounded change for the dispute flow; the release gate (`l05_dialogs`, 5 cases) tells us when it works: `lesson-05` must turn green for CONV-H01, H03–H05.

---

## Final line

```
what we traded                         5 scenarios × 5 runs before each release (165 calls) instead of 36 combinations or a daily run
what we are not doing                  the full combinatorics; two cases per conversation; language and "next day" axes;
                                       the action layer (CONV-A01) in the gate — until the confirmation fix lands
under which conditions it changes      the summarizer or prompt changes → run the dialogue set on every such change, not only before release;
                                       the D06 fix (structured case field) lands → lesson-05 must be green for CONV-H01, H03–H05;
                                       the confirmation fix lands → CONV-A01 joins the release gate
```

---

## Checklist

**After the lab**

- [x] Section 1 has the prediction
- [x] Record sheets 2.1 and 2.2 are filled in for all seven steps, with history and retrieval context
- [x] 2.3 has five runs, the range and the median
- [x] In 2.4 the moment of loss and the moment of manifestation are in different rows
- [x] 3.1 names the loss step and proves that the retrieval context at the manifestation step is complete
- [x] 3.2 classifies the self-contradiction with evidence
- [x] 4.1 has two checks with results on `clean` and `lesson-05`

**Before submitting HW_4**

- [x] Section 1 has every line filled in, including the number of cases and `set_hash`
- [x] 2.3 names the dialogue length limit
- [x] 3.3 has the conclusion of the control run, 3.4 — failures of the other scenarios or "none"
- [x] In 4.2 every case has an invariant with a ladder level and an oracle
- [x] Section 5 has three axes, at least five scenarios with roles including a control, the cost of the full combinatorics and of the plan, and the schedule
- [x] 6.1 has at least three uncovered combinations, at least one "accept"
- [x] 6.3 chooses a fix for D06 and names its cost
- [x] The final line has all three items filled in
