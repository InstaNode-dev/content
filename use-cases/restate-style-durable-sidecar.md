---
title: Restate-style durable sidecar
category: Q. Background/async agent fleets
services: ["pg"]
scenario: A Restate sidecar intercepts agent HTTP tool calls and journals them to Postgres so any crash mid-run resumes deterministically without re-charging external APIs.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Restate sidecar intercepts agent HTTP tool calls and journals them to Postgres so any crash mid-run resumes deterministically without re-charging external APIs.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Stand up a Restate-style durable sidecar in front of this agent. Every outbound tool call (HTTP POST) is journaled to Postgres before it fires; on resume after a crash, replay the journal so already-completed calls return cached responses instead of re-charging the upstream API. Provision Postgres via instanode.dev and write the sidecar in Go.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Create the journal.**

  ```sql
  CREATE TABLE journal (
    run_id UUID, step_id INT, request_hash TEXT,
    response_body JSONB, completed_at TIMESTAMPTZ,
    PRIMARY KEY (run_id, step_id)
  );
  ```

- **Step 3: Wrap every tool call in journal-first, fire-second.**

  ```go
  func DurableCall(runID string, stepID int, req *http.Request) ([]byte, error) {
      if row, ok := lookup(runID, stepID); ok { return row.Body, nil }
      resp, err := http.DefaultClient.Do(req)
      body, _ := io.ReadAll(resp.Body)
      mustExec("INSERT INTO journal VALUES ($1,$2,$3,$4,now()) ON CONFLICT DO NOTHING",
          runID, stepID, hash(req), body)
      return body, err
  }
  ```

- **Step 4: On agent restart, replay.** Load all rows for `run_id` and short-circuit each step.

- **Step 5: Idempotency key on the upstream call** prevents the rare double-charge if a crash lands between HTTP request and `INSERT`.

## Why this works on instanode.dev

Postgres provisioning in one curl makes the durable-sidecar pattern trivially deployable per-agent, not per-fleet. `ON CONFLICT DO NOTHING` plus the composite `(run_id, step_id)` key gives at-most-once semantics for every tool call — exactly the Restate guarantee — without running a separate workflow engine.

## Related cases

- [LangGraph state checkpoints](/use-cases/langgraph-state-checkpoints.md) — in-graph alternative to this out-of-process sidecar
- [Long-horizon Temporal agent workflow](/use-cases/long-horizon-temporal-agent-workflow.md) — Temporal-engine alternative when workflows last weeks
- [Form-fill state machine](/use-cases/form-fill-state-machine.md) — concrete browser-agent instance of the same crash-resume idea
