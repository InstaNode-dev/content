---
title: AI cofounder spawns a startup
category: L. Agent-factory / spawning patterns
services: ["pg", "minio", "webhook", "deploy"]
scenario: A founder-agent reads a one-line idea, provisions Postgres + S3-compatible storage + a webhook + a deploy slot, generates a landing page, and hands the bundle to a marketing sub-agent.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a founder-agent reads a one-line idea, provisions Postgres + S3-compatible storage + a webhook + a deploy slot, generates a landing page, and hands the bundle to a marketing sub-agent.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + S3-compatible storage + webhook receiver + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're an AI cofounder. Given a one-line idea ("dog-walker scheduling for NYC"), provision Postgres for bookings, S3-compatible storage for marketing assets, a webhook for Stripe/customer events, and a deploy slot for the landing page. Generate the landing page from the idea, ship it, then hand all four connection strings to a marketing sub-agent.
```

## Steps to follow

- **Step 1: Provision the full startup bundle in parallel.**

  ```bash
  curl -sX POST https://api.instanode.dev/db/new      -H "Authorization: Bearer $T" > pg.json &
  curl -sX POST https://api.instanode.dev/storage/new -H "Authorization: Bearer $T" > s3.json &
  curl -sX POST https://api.instanode.dev/webhook/new -H "Authorization: Bearer $T" > hook.json &
  wait
  ```

- **Step 2: Bootstrap the bookings schema.**

  ```sql
  CREATE TABLE bookings (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_email text,
    dog_name text,
    walk_at timestamptz,
    status text DEFAULT 'pending'
  );
  ```

- **Step 3: Generate and deploy the landing page.** Point the deploy at the bundled connection strings.

  ```bash
  curl -sX POST https://api.instanode.dev/deploy/new \
    -H "Authorization: Bearer $T" \
    -d '{
      "image": "ghcr.io/cofounder-agent/landing:latest",
      "env": {
        "DATABASE_URL": "'"$(jq -r .connection_url pg.json)"'",
        "STRIPE_WEBHOOK_URL": "'"$(jq -r .receive_url hook.json)"'"
      }
    }' | jq -r .app_url
  ```

- **Step 4: Hand the bundle to the marketing sub-agent.**

  ```json
  {
    "brand": "NYCDogWalk",
    "landing_url": "https://nycdogwalk-x7.instanode.dev",
    "events_webhook": "...",
    "assets_bucket": "instant-shared/nycdogwalk/"
  }
  ```

## Why this works on instanode.dev

Spawning a startup needs the whole bundle in one move — DB + storage + events + deploy. Anywhere else, you're juggling four signups, four billing setups, four IAM trees. Here it's four curls, all returning real resources, all wired together by the same token. The cofounder agent's TTHW (time-to-Hello-World) drops from a workday to seconds.

## Related cases

- [One-afternoon MVP backend](/use-cases/one-afternoon-mvp-backend.md) — the human-driven version of the same end-to-end ship flow
- [Pre-launch waitlist store](/use-cases/pre-launch-waitlist-store.md) — the landing-page-and-emails piece the founder agent generates
- [Side-project container deploy](/use-cases/side-project-container-deploy.md) — the bare-deploy primitive the cofounder agent calls under the hood
