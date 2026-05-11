---
title: AI cofounder spawns a startup
category: L. Agent-factory / spawning patterns
services: ["pg", "minio", "webhook", "deploy"]
scenario: A founder-agent reads a one-line idea, provisions Postgres + MinIO + a webhook + a deploy slot, generates a landing page, and hands the bundle to a marketing sub-agent.
---

## Sample agent prompt

```
You're an AI cofounder. Given a one-line idea ("dog-walker scheduling for NYC"), provision Postgres for bookings, MinIO for marketing assets, a webhook for Stripe/customer events, and a deploy slot for the landing page. Generate the landing page from the idea, ship it, then hand all four connection strings to a marketing sub-agent.
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
