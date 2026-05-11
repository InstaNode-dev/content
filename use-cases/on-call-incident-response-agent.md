---
title: On-call incident-response agent
category: F. Developer tooling
services: ["webhook", "pg"]
scenario: An on-call agent listens for alert webhooks, executes a runbook, and posts the action log back into the incident ticket.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an on-call agent listens for alert webhooks, executes a runbook, and posts the action log back into the incident ticket.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build an on-call agent that receives PagerDuty webhooks, runs the matching runbook (kubectl describe, fetch error logs, restart deployment if safe), and posts the full action log back to the incident ticket. Provision a webhook receiver and Postgres to store action logs per incident.
```

## Steps to follow

- **Step 1: Provision the webhook and DB.**

  ```bash
  curl -s -X POST https://api.instanode.dev/webhook/new | tee /tmp/wh.json
  curl -s -X POST https://api.instanode.dev/db/new
  ```

- **Step 2: Tell PagerDuty the receiver URL** from the response.

  ```bash
  jq -r .receive_url /tmp/wh.json
  # paste into PagerDuty -> Integrations -> Webhooks V3
  ```

- **Step 3: Action-log schema.**

  ```sql
  CREATE TABLE actions (
    incident_id text, step int, command text, output text, exit_code int,
    at timestamptz DEFAULT now(),
    PRIMARY KEY (incident_id, step)
  );
  ```

- **Step 4: Poller fetches buffered alerts and runs the runbook.**

  ```python
  for alert in fetch_buffered(WH_URL):
      runbook = pick_runbook(alert["service"], alert["summary"])
      for i, cmd in enumerate(runbook):
          out = run_safely(cmd, allowlist=["kubectl", "stern"])
          pg.execute("INSERT INTO actions VALUES (%s,%s,%s,%s,%s)",
                     alert["incident_id"], i, cmd, out.text, out.code)
  ```

- **Step 5: Post the log back to the ticket.**

  ```python
  log = pg.fetch("SELECT step,command,output FROM actions WHERE incident_id=%s ORDER BY step", inc)
  pagerduty.notes.create(incident_id=inc, content=render_md(log))
  ```

## Why this works on instanode.dev

The webhook receiver buffers alerts during a paging storm, so the agent processes the queue rather than hammering your runbook tooling in parallel. Action logs in Postgres give post-incident retros a clean audit trail without standing up a separate logging stack.

## Related cases

- [Deploy-status MCP server](/use-cases/deploy-status-mcp-server.md) — live status the on-call agent reads while running a runbook
- [Slack/Discord async bot factory](/use-cases/slack-discord-async-bot-factory.md) — another webhook-driven durable bot pattern
- [CI flake-tracker](/use-cases/ci-flake-tracker.md) — consumes the alerts that flaky-test detection often generates
