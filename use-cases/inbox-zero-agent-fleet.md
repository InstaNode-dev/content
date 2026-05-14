---
title: Inbox-zero agent fleet
category: Q. Background/async agent fleets
services: ["minio", "pg", "webhook"]
scenario: A Cloudflare-Email-style fleet runs one durable agent per user that triages incoming email, files attachments in S3-compatible storage, and stores extracted action items in Postgres.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Cloudflare-Email-style fleet runs one durable agent per user that triages incoming email, files attachments in S3-compatible storage, and stores extracted action items in Postgres.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (S3-compatible storage + Postgres + webhook receiver) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a per-user durable email-triage agent. For each user, provision a S3-compatible bucket for attachments, a Postgres row-store for extracted action items, and a webhook endpoint that Cloudflare Email Workers will POST inbound messages to. Each agent should classify, file attachments, and persist next-actions. Return the three URLs and the user-id mapping.
```

## Steps to follow

- **Step 1: Provision the Postgres for action items.**

  ```bash
  curl -s -X POST https://api.instanode.dev/db/new | tee /tmp/pg.json
  ```

- **Step 2: Provision the attachment bucket.**

  ```bash
  curl -s -X POST https://api.instanode.dev/storage/new | tee /tmp/minio.json
  ```

- **Step 3: Provision an inbound-email webhook receiver.**

  ```bash
  curl -s -X POST https://api.instanode.dev/webhook/new | tee /tmp/wh.json
  ```

- **Step 4: Create the action-items schema.**

  ```sql
  CREATE TABLE actions (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id text NOT NULL,
    message_id text NOT NULL,
    due_at timestamptz,
    body text,
    created_at timestamptz DEFAULT now()
  );
  CREATE INDEX ON actions (user_id, due_at);
  ```

- **Step 5: Worker reads inbound webhook batches.**

  ```python
  inbound = requests.get(f"{WH_URL}/requests?since={cursor}").json()
  for msg in inbound:
      items = triage_agent.run(msg["body"])
      pg.executemany("INSERT INTO actions(...) VALUES (...)", items)
      for att in msg["attachments"]:
          s3.put_object(Bucket="inbox", Key=f"{user}/{att['name']}", Body=att["bytes"])
  ```

## Why this works on instanode.dev

The webhook receiver buffers inbound mail for you, so the triage agent can fall behind during a spike without losing messages. Postgres and S3-compatible storage live under the same token, which means one billing relationship and one upgrade path when a user's fleet grows past the hobby quota.

## Related cases

- [Slack/Discord async bot factory](/use-cases/slack-discord-async-bot-factory.md) — chat-platform sibling of this per-user durable-agent pattern
- [Overnight dossier fleet](/use-cases/overnight-dossier-fleet.md) — another async fleet writing Mongo + S3-compatible storage artifacts
- [Per-tenant chatbot factory at signup](/use-cases/per-tenant-chatbot-factory-at-signup.md) — tenant-spawning variant of the same per-user idea
