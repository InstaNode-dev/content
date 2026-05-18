---
title: Solo-founder analytics warehouse
category: H. Indie & SaaS founders
services: ["pg"]
scenario: A founder pipes product events into Postgres and runs SQL queries via an MCP-Postgres tool from chat.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a founder pipes product events into Postgres and runs SQL queries via an MCP-Postgres tool from chat.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
I'm a solo founder. Provision a Postgres on instanode.dev, wire my Next.js app to fire-and-forget events into a wide `events` table, and let me query "MAU last 30 days, retention curves, top features" from chat via the MCP-Postgres tool.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"solo-founder-analytics-warehouse-db"}' | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Wide events table — no schema migrations ever.**

  ```sql
  CREATE TABLE events (
    id BIGSERIAL, user_id TEXT, name TEXT,
    props JSONB, ts TIMESTAMPTZ DEFAULT now()
  );
  CREATE INDEX ON events (name, ts DESC);
  CREATE INDEX ON events USING gin (props);
  ```

- **Step 3: Fire-and-forget from app.**

  ```typescript
  await fetch("/api/track", {
    method: "POST",
    body: JSON.stringify({ user_id, name: "feature_used", props: { feature: "export" } }),
  });
  ```

- **Step 4: Configure MCP-Postgres.**

  ```json
  { "mcpServers": { "postgres": { "command": "mcp-postgres", "env": { "DATABASE_URL": "..." } } } }
  ```

- **Step 5: Ask from chat.**

  ```sql
  SELECT date_trunc('day', ts), count(DISTINCT user_id)
  FROM events WHERE ts > now() - interval '30 days'
  GROUP BY 1 ORDER BY 1;
  ```

## Why this works on instanode.dev

A solo founder doesn't need PostHog, Mixpanel, and Snowflake; they need one place to ask SQL questions and a real Postgres on `MCP-Postgres` answers everything. `gin` index on the props JSONB makes "filter by any nested key" cheap without a star schema.

## Related cases

- [Pre-launch waitlist store](/use-cases/pre-launch-waitlist-store.md) — an upstream funnel source that lands in this warehouse
- [Stripe-event entitlements](/use-cases/stripe-event-entitlements.md) — subscription-state source for cohort analysis in the warehouse
- [Scraped product-price history](/use-cases/scraped-product-price-history.md) — another time-series Postgres workload for indie founders
