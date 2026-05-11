---
title: Copy-on-write Postgres for agent migrations
category: K. Ephemeral agent runtimes
services: ["pg"]
scenario: An Ardent-style coding agent forks a 6-second copy-on-write clone of prod Postgres, runs the candidate ALTER TABLE, validates row counts, then destroys the clone.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an Ardent-style coding agent forks a 6-second copy-on-write clone of prod Postgres, runs the candidate ALTER TABLE, validates row counts, then destroys the clone.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're an Ardent-style migration agent. Take a snapshot of prod Postgres, fork a copy-on-write clone via instanode.dev, run the candidate ALTER TABLE on the clone, compare row counts and constraints against the source, and if all checks pass print the migration SQL. Destroy the clone when done.
```

## Steps to follow

- **Step 1: Provision a scratch Postgres clone.** The fork is a fresh shared Postgres seeded from a dump.

  ```bash
  CLONE_URL=$(curl -sX POST https://api.instanode.dev/db/new \
    -H "Content-Type: application/json" \
    -d '{"seed_dump_url":"https://my-bucket.s3.amazonaws.com/prod-snapshot.sql.gz"}' \
    | jq -r .connection_url)
  ```

- **Step 2: Run the candidate migration on the clone.** Wrap in a transaction for easy rollback while inspecting.

  ```sql
  BEGIN;
  ALTER TABLE orders ADD COLUMN refunded_at timestamptz;
  UPDATE orders SET refunded_at = now() WHERE status = 'refunded';
  ```

- **Step 3: Validate against expectations.** Row counts, null counts, constraint violations.

  ```sql
  SELECT COUNT(*) AS total,
         COUNT(refunded_at) AS filled,
         COUNT(*) FILTER (WHERE status='refunded' AND refunded_at IS NULL) AS broken
  FROM orders;
  ```

- **Step 4: Destroy the clone.** Resources expire at 24h on free tier, or destroy immediately via API.

  ```bash
  curl -sX DELETE https://api.instanode.dev/db/$DB_TOKEN
  ```

## Why this works on instanode.dev

Provisioning is synchronous and sub-second — the agent gets a usable database in the same request, not a pending status it has to poll. Ephemeral anonymous resources auto-reap at 24h, so a forgotten clone has bounded blast radius. Pair it with a destroy call after a green run, and you get the safety properties of a real fork without standing up pg_basebackup infrastructure.

## Related cases

- [Ephemeral test database for a risky migration](/use-cases/ephemeral-test-database-for-a-risky-migration.md) — the slower-fork baseline this CoW pattern accelerates
- [Parallel SQL-plan probe](/use-cases/parallel-sql-plan-probe.md) — uses forked Postgres clones in parallel for EXPLAIN probing
- [Devin-style PR-bot fleet](/use-cases/devin-style-pr-bot-fleet.md) — wraps this clone-and-test loop in a per-issue agent worker
