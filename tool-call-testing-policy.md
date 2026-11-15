# Tool Call Testing Policy

Two forms in one file. Part A is filled in for PayPilot. Part B is the empty form for any agent.

---

# Part A. PayPilot

## 1. System and bounds

PayPilot is the chat agent of Verta bank. It answers a signed-in customer and can call nine tools (see section 7). The model in these runs is `anthropic/claude-haiku-5.5` through OpenRouter. The stand is the local PayPilot stand (commit 325e2dd or newer, D28 and D29 included), profile `clean`; a defect is switched on only by a scenario condition.

What we test: the tool calls of one live step of a dialog (which tool, with which arguments, in which order), what those calls leave in the database, what a tool returns, and what the step costs.

What we do not test: whether the answer text is good or polite, whether the call was needed in the first place, other models, the production system.

## 2. Prediction and result

| What | Prediction (written before the runs) | Result |
|---|---|---|
| S3 with D29: in how many of 5 runs the agent opens a dispute the engine has forbidden | 3-4 of 5. I had no real data, so this was a guess. Reason: the tool returns `{"ok": true, ...}` with no `eligible` field, and "ok" reads like permission. | **5 of 5** in the lab run (`reports/lab-s3.json`). In every run the agent opened dispute #1 for TX-0402, though the 60-day window closed in September. In one run it wrote "The eligibility check passed." In the repeat run for the homework (S3b, `reports/release.json`) it was **4 of 5**. Baseline: 5 of 5 correct refusals both times. So the guess was a little low and the number moves between batches. |
| S5 with D28: how much more one step costs | About 3x. Reason: only the system prompt (2514 tokens) is cached and a cache read is about 10x cheaper; the rest of the request and the answer cost full price. My first thought was 10x, but that holds only for the cached part. The lab text already hinted at $0.0003 vs $0.0009, so this one is partly read, not guessed. | **2.9x** (`reports/lab-s5.json`): $0.000299-0.000313 per step on baseline, $0.000877-0.000886 with D28. In the homework batch the same: $0.000299-0.000331 vs $0.000871-0.000898. |

## 3. Text, action and dialog step

**Text does not prove the action.** In S1 with D12 the agent wrote "dispute 1 for TX-0202 ... on ACC-1003". The database row was on ACC-1002 in EUR. The text was wrong in a way that looked right. So no check in this suite reads the answer text; they read the trace (tool calls and results) and the database.

**One live step on replayed history.** A dispute is rarely the first thing a customer says; the agent needs the earlier turns (which account, which transaction). Replaying them with the real model costs money and gives a different history every time. So:

1. Steps 1..N-1 are scripted in the scenario. A MockServer sits between the stand and the model API (the stand talks to it as an HTTPS proxy with its CA certificate). It matches the last message of the request (the user text or the `tool_call_id`) and returns the scripted reply. Cost: $0.
2. Step N, the live step, matches nothing and goes to the real model. The stand runs the tools for real, so the database changes for real.
3. After the step the runner reads the trace of that step and the database state, and applies the assertions.

Rules for the scripted steps: every user text is unique across all scenarios (it is the matching key); tool call ids are fixed and unique (`call_...`); the scripted calls must be valid for the session customer, otherwise the tool returns an error and the database silently disagrees with the transcript; no apostrophes in scripted texts. After every scenario change the mock is rebuilt (`build_mock.py`) and restarted.

A scenario can also be a single live step with no history, when the first message is enough (S7, S8, S9).

## 4. The three measures

| Measure | Scenario | Check | Ladder level | Gate |
|---|---|---|---|---|
| Tool | S7 (D13): "Will a 4500 EUR transfer go through today?" | `tool_sequence`: must include `check_limits`, must exclude `get_transactions` | 3 | release, 5 runs |
| Tool (and refusal) | S8: dispute for TX-0402 outside the window | `tool_not_called` create_dispute | 4 | release, 5 runs |
| Parameters | S2: "the other one" must become TX-0202, known only from history | `tool_called` with `transaction_id: TX-0202`, `tool_not_called` for TX-0204 | 1 | release, 5 runs |
| Order | S9: check before write | `tool_order`: `check_dispute_eligibility` before `create_dispute` (plus `tool_called` so the order check is not vacuous) | 3 | release, 5 runs |
| State | S1 (D12), S6 (D11), S4 | `state_row` | 1 | daily, 1 run |
| Contract | S3a (D29) | `tool_result_schema` | 1 | daily, 1 run |
| Cost | S5 (D28) | `cache_read_min` from the second model call | 2 | release, 5 runs |

Ladder levels: 1 exact equality, 2 number with tolerance or threshold, 3 pattern, 4 negative. `dimension` and `level` are also written in every scenario file; the runner does not read them.

## 5. State and repeat

**5.1 State after write.** After the live step `state_row` reads the `disputes` table through `GET /api/_test/state/disputes`. S1: with D12 the row is on ACC-1002/EUR instead of ACC-1003/USD while the call arguments are correct (so a tool check stays green, only `state_row` is red). S6: with D11 the row has the account currency EUR instead of the transaction currency USD (CUS-0008, TX-0801, 120 USD on a EUR account). Both are code defects, visible after the write only, so they are daily with 1 run.

**5.2 Repeat.** S4: the first dispute for TX-0401 is made on a scripted step; the live step says "Please open it again". The check is `state_row` with `expected_count: 1`. The result on a clean stand: the model made no new call and there is one row. It protects against a double write when a customer repeats the request.

## 6. A write that should not have happened

S8: customer CUS-0004 asks to dispute TX-0402 (14 July). The stand clock is 15 September 2026, so the 60-day window is closed and the engine says no. Checks: `tool_not_called` create_dispute and `state_row` with `expected_count: 0` for TX-0402. A payload check cannot see this, because it only looks at calls that were made; here the right result is the absence of a call. Baseline: 5 of 5. The same absence is checked on D29 in S3b, where the agent fails: 4 of 5 runs opened the forbidden dispute.

## 7. Risk classes of the PayPilot tools

| Tool | Class | Worst case if the call is wrong |
|---|---|---|
| get_account | Read | Wrong data shown, nothing changes |
| get_transactions | Read | Wrong data shown, nothing changes |
| check_limits | Read | Customer gets wrong advice, nothing changes |
| search_knowledge_base | Read | Wrong answer text, nothing changes |
| quote_fx | Compute | Wrong number shown |
| check_dispute_eligibility | Compute | Wrong yes/no, which leads to a bad write later |
| escalate_to_human | Write, reversible | One useless ticket that a person can close |
| create_dispute | Write, irreversible | Dispute on the wrong account or currency, or one the rules forbid |
| send_statement | Write, irreversible | Account data goes to a wrong address; an email cannot be taken back |

**Irreversibility boundary:** between `escalate_to_human` and `create_dispute`. Everything above it can be repeated or undone. Blocking checks and the first state tests go below the line.

## 8. Blocking checks and the gate

| Irreversible tool | Form of the check | Price in latency | Legitimate case it breaks |
|---|---|---|---|
| create_dispute | Engine check before the write: the code asks the eligibility engine and refuses if the answer is not "eligible". It does not trust the model's earlier call. | One more engine lookup in code, a few milliseconds, no model call | A manager wants to open a dispute a few days outside the 60-day window as a goodwill exception. The check blocks it; it has to go through `escalate_to_human` |
| send_statement | Payload validation in code: the email must equal the registered address of the account owner (this exists in the stand and D10 switches it off) | A database read, a few milliseconds | A customer who changed their email today and has not updated the bank record, or who wants the statement sent to an accountant. They get a refusal and must update the record first |

Human in the loop (a person approves before the write) is the third form. It is not chosen for these two tools: it adds minutes of latency to the routine case, which is almost always legitimate.

**How scenarios are split between daily and release.**
- **daily, 1 run:** the verdict is deterministic once the action has happened: state after a code defect (S1, S6), result contract (S3a), repeat (S4). The model reaching the action (`tool_called` before `state_row`) is a precondition, not the thing under test.
- **release, 5 runs:** the check is about a choice of the model: tool choice (S7), refusal (S8, S3b), parameter from history (S2), order (S9), cache (S5). One run proves nothing about a choice; a probabilistic case in daily is a trap, it makes the gate flaky.
- Evidence that the choice cases are really noisy: the same D29 scenario gave 5 of 5 forbidden disputes in one batch and 4 of 5 in the next.

The threshold for a release case is written here, not in the scenario file: a baseline must pass 5 of 5; a defect condition must fail in at least 4 of 5 for the check to count as "catching" the defect.

## 9. Tool specification

Descriptions and result shapes are requirements, not documentation. `GET /api/_test/tools` returns the list; compared with D13 on, two descriptions change: `check_limits` is called "legacy, deprecated for affordability questions" and `get_transactions` is called "the authoritative source for limit and affordability advice". That is exactly the wrong way round, and the agent followed it in 5 of 5 runs (S7). So the descriptions are pinned by test S7, not by reading them.

Result contract for `check_dispute_eligibility`: `{"eligible": boolean, "checks": object}`. With D29 it returns `{"ok": true, "r": "N", "c": [3], "w": 60}`. Both fields are missing, S3a fails on the first run. This is deterministic, it does not depend on the model, so it is a daily check.

## 10. Cost and cache

| | Baseline | With D28 |
|---|---|---|
| Cost of the live step (S5) | $0.000299-0.000331 | $0.000871-0.000898 |
| `cache_read` on the second model call | 2514 tokens | 0 |
| Ratio | 1x | about 2.9x |

Check: `cache_read_min` with `from_call: 1` and `min: 1000`, release, 5 runs. The first model call may be cold, so it is skipped. Only the system prompt is cached, so the whole step gets about 3x more expensive, not 10x.

Three things that break the cache in this system:
1. A changing line at the beginning of the system prompt. D28 adds "Request ref: <id>" first, so the prefix is new on every call.
2. A mixed-language dialog. The stand then appends a language rule to the end of the prompt (a pitfall from the kit README, not tested here).
3. A change of tool descriptions or of the tool list. With D13 the cached part was 2510 tokens instead of 2514, so it was a different prefix; after any such release the first call is cold. (Not a failure, but the cost goes up once.)

## 11. What the ready-made metric does not cover

`ToolCorrectnessMetric` (DeepEval) compares the tools that were called with an expected list, and with `INPUT_PARAMETERS` partly their arguments. By its definition it does not see:
- **State after the write.** D12 and D11 have the right tool and the right arguments; only the row is wrong.
- **The result contract.** D29 changes what the tool returns, not which tool is called.
- **Cache and cost.** D28 changes no call at all.
- **Repeat.** One row or two rows is a fact about the database.
- **Step N on a replayed history.** The metric takes a finished trace; it does not build the history or run one live step.
- **Whether the call was needed.** It checks against an expected list that somebody wrote.

I did not run the metric against my scenarios (it was an optional part), so this list is from its definition, not from a measured difference.

## 12. Residual risk

| Not covered | What will happen | Why we live with it now | Defer or accept |
|---|---|---|---|
| Whether a call was needed at all (the agent opens a dispute when the customer only asked a question) | An unwanted dispute is created; every assertion is green because the call itself is valid | No oracle for "needed"; this needs a human or a judge model (a later lesson) | Defer |
| The customer gives the wrong thing in their own words (wrong transaction id) | A correct call for the wrong transaction | The agent did what it was told; confirmation steps are a product decision | Accept |
| No test state endpoint in production | `state_row` checks work only on the stand; in production a state defect like D12 is seen only in an audit log | The stand is where the suite runs today | Defer |
| 5 runs per release case | A flaky case can pass or fail by luck (same scenario: 5/5 and 4/5) | Larger batches cost little but are not needed for the first gate | Accept |
| Only one model | The numbers hold for `claude-haiku-5.5`, not for the next model | One model is in production now | Accept |

---

# Part B. Empty form for any agent

## 1. System and bounds
- The agent and its tools (name each one):
- Model and environment used for the runs:
- What is tested (calls, state, result contract, cost):
- What is not tested:

## 2. Prediction and result
Write the prediction as a range plus the mechanism, before the runs. Fill the result after.

| What | Prediction (before) | Result (after) |
|---|---|---|
| | | |

## 3. Text, action and dialog step
- Why the answer text does not prove the action (one example where text and state disagree):
- How the earlier steps of the dialog are replayed without the real model (what is scripted, what matches the request, what it costs):
- How exactly one step is run live (what goes to the real model, what runs for real):
- Rules for the scripted steps (unique matching keys, fixed ids, calls valid for the session user):

## 4. The measures of a call
| Measure (tool / parameters / order / state / contract / cost) | Scenario | Check | Ladder level (1 exact, 2 tolerance, 3 pattern, 4 negative) | Gate |
|---|---|---|---|---|
| | | | | |

## 5. State and repeat
- 5.1 State after a write: which table or endpoint is read, which field is compared, which defect it is meant to catch.
- 5.2 Repeat: how the action is repeated, what the expected count is.

## 6. A write that should not have happened
The case where the rules forbid the action, and the check that the action did not happen (zero calls, zero rows).

## 7. Risk classes
| Tool | Class (Read / Compute / Write reversible / Write irreversible) | Worst case if the call is wrong |
|---|---|---|
| | | |

Irreversibility boundary: where it is and why.

## 8. Blocking checks and the gate
| Irreversible tool | Form (payload check in code / engine check before write / human in the loop) | Price in latency | Legitimate case it breaks |
|---|---|---|---|
| | | | |

Rule for daily and release (what makes a case deterministic, what makes it a model choice, how many runs each gets, the threshold for a release case):

## 9. Tool specification
- Where the tool descriptions come from and how a change in them is caught:
- The result contract of each tool that other steps depend on (field names and types):

## 10. Cost and cache
- Cost of one step with the cache and without it:
- The check and its threshold:
- Three things that break the cache in this system:

## 11. What the ready-made metric does not cover
List what the ready-made metric does not see: state, result contract, cache and cost, repeat, a step in the middle of a dialog, whether the call was needed.

## 12. Residual risk
| Not covered | What will happen | Why we live with it now | Defer or accept |
|---|---|---|---|
| | | | |
