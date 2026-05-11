---
title: Solo-founder analytics warehouse
category: H. Indie & SaaS founders
services: ["pg"]
scenario: A founder pipes product events into Postgres and runs SQL queries via an MCP-Postgres tool from chat.
---

## Sample agent prompt

```
I'm a solo founder. Provision a Postgres on instanode.dev, wire my Next.js app to fire-and-forget events into a wide `events` table, and let me query "MAU last 30 days, retention curves, top features" from chat via the MCP-Postgres tool.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new | tee db.json
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
