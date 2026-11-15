# Tool Call Testing Policy · PayPilot

Work in progress. Sections 2 and 7 are filled so far. Predictions in section 2 were written before the lab runs.

## 2. Prediction and result

| What | Prediction | Result |
|---|---|---|
| S3 with D29: in how many of 5 runs the agent opens a dispute the engine has forbidden | 3-4 of 5. I have no real data, so this is a guess. Reason: the tool returns `{"ok": true, ...}` with no `eligible` field, and "ok" reads like permission. Sometimes the agent will go and look for an explanation instead. | **5 of 5** (reports/lab-s3.json). The guess was a bit low. In every run the agent opened dispute #1 for TX-0402, although the 60-day window closed on 12 September. In one run it even wrote "The eligibility check passed." Baseline: 5 of 5 runs refused correctly. `tool_result_schema` was red in all 5 defect runs, whatever the agent did. |
| S5 with D28: how much more one step costs | about 3x. Reason: only the system prompt (2514 tokens) is cached, and a cache read is about 10x cheaper. The rest of the request and the answer cost full price, and a cache write adds a surcharge. My first thought was 10x, but that holds only for the cached part. Note: the lab text already gives about $0.0003 vs $0.0009, so this one is partly read, not guessed. | **2.9x** (reports/lab-s5.json). Baseline $0.000299-0.000313 per step, cache_read [2514, 2514]. With D28 $0.000877-0.000886, cache_read [0, 0]. The same 5 of 5 on both sides, so no run was a lucky cache hit. |

## 7. Risk classes of the PayPilot tools

| Tool | Class | Why | Worst case if the call is wrong |
|---|---|---|---|
| get_account | Read | Only reads the customer's accounts | Wrong data shown, nothing changes |
| get_transactions | Read | Only reads | Wrong data shown, nothing changes |
| check_limits | Read | Only reads limits | Customer gets wrong advice, nothing changes |
| search_knowledge_base | Read | Only reads articles | Wrong answer text, nothing changes |
| quote_fx | Compute | Calculates a quote, stores nothing | Wrong number shown |
| check_dispute_eligibility | Compute | Asks the engine, stores nothing | Wrong yes/no, which can lead to a bad write later |
| escalate_to_human | Write, reversible | Creates a ticket a person can close | One useless ticket |
| create_dispute | Write, irreversible | Creates a dispute row and starts a process at the bank | Dispute on the wrong account, wrong currency, or one that the rules forbid |
| send_statement | Write, irreversible | Sends an email, and an email cannot be taken back | Account data goes to the wrong address |

**Irreversibility boundary.** It sits between escalate_to_human and create_dispute. Everything above it can be repeated or undone. create_dispute and send_statement cannot, so the blocking checks and the daily tests on database state go there first.
