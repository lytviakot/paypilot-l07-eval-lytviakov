# PayPilot tool call tests (L07 homework)

A copy of the L07 kit with my scenarios and my Tool Call Testing Policy.

- `scenarios/` ten scenarios (S1, S2, S3a, S3b, S4-S9). S6, S7, S8, S9 are my own.
- `tool-call-testing-policy.md` the Policy, filled in for PayPilot (Part A) and an empty form (Part B).
- `reports/` raw run reports: `lab-*.json` (lab), `daily.json`, `baseline.json`, `release.json` (homework).

## Run

The stand runs behind the MockServer proxy (stand started with `mock/stand.override.yml`, `OPENROUTER_API_KEY` in the stand `.env`; no key is stored here).

```bash
docker compose up -d llm-mock
docker compose run --rm eval --gate daily                 # 4 scenarios, 1 run
docker compose run --rm eval --condition baseline         # every baseline
docker compose run --rm eval --gate release --runs 5      # model-choice scenarios
```

After any change in `scenarios/`:

```bash
docker compose run --rm --entrypoint python eval build_mock.py && docker compose restart llm-mock
```

## Gate split

| gate | runs | scenarios |
|---|---|---|
| daily | 1 | S1, S3a, S4, S6 |
| release | 5 | S2, S3b, S5, S7, S8, S9 |
