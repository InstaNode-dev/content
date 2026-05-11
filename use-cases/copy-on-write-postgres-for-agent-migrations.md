---
title: Copy-on-write Postgres for agent migrations
category: K. Ephemeral agent runtimes
services: ["pg"]
scenario: An Ardent-style coding agent forks a 6-second copy-on-write clone of prod Postgres, runs the candidate ALTER TABLE, validates row counts, then destroys the clone.
---

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
