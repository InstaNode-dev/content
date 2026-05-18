---
title: CI flake-tracker
category: F. Developer tooling
services: ["webhook", "pg"]
scenario: A CI bot ingests test-run webhooks, fingerprints failures, and surfaces "this test is flaky 30% of runs".
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a CI bot ingests test-run webhooks, fingerprints failures, and surfaces "this test is flaky 30% of runs".

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're a CI flakiness tracker. Receive GitHub Actions / CircleCI webhooks on every test run. Fingerprint failures by (test_name + sha256(stack_trace_first_line)). Store run history in Postgres. Compute "flaky" as "passed in some runs, failed in others within the last 7 days" and surface a daily list.
```

## Steps to follow

- **Step 1: Provision webhook + Postgres.**

  ```bash
  HOOK=$(curl -sX POST https://api.instanode.dev/webhook/new -H 'Content-Type: application/json' -d '{"name":"ci-flake-tracker-webhook"}' -H "Authorization: Bearer $T" | jq -r .receive_url)
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"ci-flake-tracker-db"}'   -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Configure CI to POST results to `$HOOK`.**

  ```yaml
  # .github/workflows/test.yml
  - name: Report results
    if: always()
    run: |
      curl -X POST "$WEBHOOK" -H 'content-type: application/json' \
        -d "$(jq -n --arg sha "$GITHUB_SHA" --slurpfile r results.json \
              '{sha:$sha, tests:$r[0]}')"
  ```

- **Step 3: Schema with fingerprint.**

  ```sql
  CREATE TABLE test_runs (
    id bigserial PRIMARY KEY,
    commit_sha text NOT NULL,
    test_name text NOT NULL,
    fingerprint text,
    status text CHECK (status IN ('pass','fail','skip')),
    ran_at timestamptz DEFAULT now()
  );
  CREATE INDEX idx_test_recent ON test_runs (test_name, ran_at DESC);
  ```

- **Step 4: Flake query — pass-AND-fail within 7d.**

  ```sql
  SELECT test_name,
         COUNT(*) FILTER (WHERE status='fail')::float / COUNT(*) AS fail_rate,
         COUNT(*) AS n
  FROM test_runs
  WHERE ran_at > now() - interval '7 days'
  GROUP BY test_name
  HAVING COUNT(DISTINCT status) > 1 AND COUNT(*) > 10
  ORDER BY fail_rate DESC LIMIT 20;
  ```

## Why this works on instanode.dev

Flake tracking is two pieces: an HTTP sink for CI to POST to, and SQL to compute pass/fail variance. The webhook URL is already a public endpoint with stored history — your CI doesn't need to know about your DB. The SQL is bog-standard but needs real indexes (the HAVING DISTINCT trick is slow on a non-indexed test_name column). Two curls, no Datadog bill.

## Related cases

- [SARIF scan-result store](/use-cases/sarif-scan-result-store.md) — another CI-side Postgres warehouse for tracking drift over time
- [PR-review bot triggered by webhooks](/use-cases/pr-review-bot-triggered-by-webhooks.md) — consumes the same kind of GitHub webhook firehose
- [High-volume PR-review pipeline](/use-cases/high-volume-pr-review-pipeline.md) — the at-scale version of the same CI-feedback loop
