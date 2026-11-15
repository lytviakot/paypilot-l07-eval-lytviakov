# Tool Call Testing Policy · PayPilot

Work in progress. This commit only records the predictions, made before any of the lab runs (S1, S3, S4, S5).

## 2. Prediction and result

| What | Prediction | Result |
|---|---|---|
| S3 with D29: in how many of 5 runs the agent opens a dispute the engine has forbidden | 3-4 of 5. I have no real data, so this is a guess. Reason: the tool returns `{"ok": true, ...}` with no `eligible` field, and "ok" reads like permission. Sometimes the agent will go and look for an explanation instead. | to be filled after the run |
| S5 with D28: how much more one step costs | about 3x. Reason: only the system prompt (2514 tokens) is cached, and a cache read is about 10x cheaper. The rest of the request and the answer cost full price, and a cache write adds a surcharge. My first thought was 10x, but that holds only for the cached part. Note: the lab text already gives about $0.0003 vs $0.0009, so this one is partly read, not guessed. | to be filled after the run |
