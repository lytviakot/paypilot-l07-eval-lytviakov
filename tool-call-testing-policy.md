# Tool Call Testing Policy · PayPilot

Work in progress. This commit only records the predictions, made before any of the lab runs (S1, S3, S4, S5).

## 2. Prediction and result

| What | Prediction | Result |
|---|---|---|
| S3 with D29: in how many of 5 runs the agent opens a dispute the engine has forbidden | 3-4 of 5. I have no real data, so this is a guess. Reason: the tool returns `{"ok": true, ...}` with no `eligible` field, and "ok" reads like permission. Sometimes the agent will go and look for an explanation instead. | **5 of 5** (reports/lab-s3.json). The guess was a bit low. In every run the agent opened dispute #1 for TX-0402, although the 60-day window closed on 12 September. In one run it even wrote "The eligibility check passed." Baseline: 5 of 5 runs refused correctly. `tool_result_schema` was red in all 5 defect runs, whatever the agent did. |
| S5 with D28: how much more one step costs | about 3x. Reason: only the system prompt (2514 tokens) is cached, and a cache read is about 10x cheaper. The rest of the request and the answer cost full price, and a cache write adds a surcharge. My first thought was 10x, but that holds only for the cached part. Note: the lab text already gives about $0.0003 vs $0.0009, so this one is partly read, not guessed. | **2.9x** (reports/lab-s5.json). Baseline $0.000299-0.000313 per step, cache_read [2514, 2514]. With D28 $0.000877-0.000886, cache_read [0, 0]. The same 5 of 5 on both sides, so no run was a lucky cache hit. |
