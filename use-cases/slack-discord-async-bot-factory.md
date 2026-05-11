---
title: Slack/Discord async bot factory
category: Q. Background/async agent fleets
services: ["webhook", "pg"]
scenario: A factory spins up one durable agent per Slack workspace that subscribes to events via webhook, batches them, and runs nightly summarizer jobs persisting digests in Postgres.
---

## Sample agent prompt

```
Stand up a per-workspace Slack bot factory. For each tenant: provision a webhook URL via instanode.dev to receive Slack events, persist them into a per-tenant rows in Postgres, and at 02:00 UTC run a summarizer agent that produces a digest message back into Slack. Use one shared Postgres for all tenants but namespace rows by tenant_id.
```

## Steps to follow

- **Step 1: Provision shared Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Per-tenant webhook on workspace install.**

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new)
  echo "$WH" > tenants/$WORKSPACE_ID-webhook.json
  # Register receive_url with Slack as the event endpoint
  ```

- **Step 3: Drain events into Postgres.**

  ```python
  events = requests.get(f"{webhook_base}/{token}/requests").json()
  for e in events:
      conn.execute("INSERT INTO slack_events(tenant_id,payload,ts) VALUES (%s,%s,%s)",
                   (workspace_id, json.dumps(e["body"]), e["received_at"]))
  ```

- **Step 4: Nightly summarizer per tenant.**

  ```sql
  SELECT payload->>'text' FROM slack_events
  WHERE tenant_id = $1 AND ts > now() - interval '24h';
  ```

- **Step 5: Post the digest** back via the workspace's bot token.

## Why this works on instanode.dev

`/webhook/new` is essentially "give me an HTTPS URL that captures every POST" — perfect for Slack/Discord event subscribe endpoints without writing a public listener service per tenant. One curl per tenant onboarded; replay history is queryable via the requests endpoint.
