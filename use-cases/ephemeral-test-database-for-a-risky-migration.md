---
title: Ephemeral test database for a risky migration
category: A. AI coding agents
services: ["pg"]
scenario: A coding agent provisions a throwaway Postgres in under a second, dry-runs a destructive migration, inspects the result, then discards the database — no production blast radius.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a coding agent provisions a throwaway Postgres in under a second, dry-runs a destructive migration, inspects the result, then discards the database — no production blast radius.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
The migration in migrations/0047_drop_legacy_email.sql does ALTER TABLE orders DROP COLUMN customer_email and it's blocking the release. Our prod orders table is 84M rows and three downstream services may still read that column. Claim a fresh Postgres on instanode.dev, pg_restore last night's dump (~/backups/prod-2026-05-10.dump, 1.2GB) into it, run the migration inside a transaction, query pg_depend for any view/trigger/index that still depends on the column, then ROLLBACK and tell me whether it's safe to ship — or which references to fix first.
```

## Steps to follow

The example below threads a single migration — dropping `orders.customer_email` after the email moved to a `customers` table — across every step. By the end you'll know whether the drop is safe to merge.

- **Step 1: Claim a throwaway Postgres.** One POST, no signup. The token auto-reaps in 24 hours, so a forgotten test DB never lingers on your bill.

  ```bash
  RESP=$(curl -sX POST https://api.instanode.dev/db/new)
  export PG_URL=$(echo "$RESP" | jq -r .connection_url)
  export PG_TOKEN=$(echo "$RESP" | jq -r .token)
  echo "throwaway db ready: ${PG_URL%@*}@***"
  ```

- **Step 2: Restore last night's prod dump.** A real Postgres accepts `pg_restore` directly — no SQLite stand-in, no Docker image to build.

  ```bash
  pg_restore --no-owner --no-acl -j 4 -d "$PG_URL" ~/backups/prod-2026-05-10.dump
  psql "$PG_URL" -c "SELECT count(*) AS rows FROM orders;"
  # rows
  # ---------
  #  84112903
  ```

- **Step 3: Run the migration inside a transaction.** `BEGIN ... ROLLBACK` lets the agent observe the post-migration state without committing.

  ```sql
  BEGIN;
  ALTER TABLE orders DROP COLUMN customer_email;

  -- any view, trigger, index, or default that referenced the column?
  SELECT n.nspname AS schema,
         c.relname AS object,
         c.relkind AS kind
    FROM pg_depend d
    JOIN pg_class c   ON d.objid = c.oid
    JOIN pg_namespace n ON c.relnamespace = n.oid
   WHERE d.refobjid = 'orders'::regclass
     AND d.deptype = 'n';
  ROLLBACK;
  ```

- **Step 4: Cross-check the application code.** `pg_depend` won't catch raw SQL string references in Python or Go, so the agent greps the working tree too.

  ```bash
  rg -n --type-add 'sql:*.sql' -t sql -t py -t go -t ts 'customer_email' src/ tests/ migrations/ \
    | grep -v '^migrations/0047_' \
    | tee /tmp/refs.txt
  echo "remaining references: $(wc -l < /tmp/refs.txt)"
  ```

- **Step 5: Verdict + cleanup.** The agent reports back, then deletes the throwaway (or just lets it expire).

  ```bash
  if [ -s /tmp/refs.txt ]; then
    echo "UNSAFE — $(wc -l < /tmp/refs.txt) code references remain. See /tmp/refs.txt"
  else
    echo "SAFE — pg_depend clean, no string references in tree."
  fi
  curl -sX DELETE "https://api.instanode.dev/db/$PG_TOKEN"
  # {"ok":true,"deleted":"<uuid>"}
  ```

## Why this works on instanode.dev

The usual options for "test a destructive migration" all fail for an agent. Spinning up RDS or Cloud SQL takes 4–7 minutes and a service account. **Ardent** and **Neon branches** are fast but require an auth handshake the agent doesn't have credentials for — and Ardent's $$$/hour per-branch fee makes 50 throwaways per day genuinely expensive. **Supabase branches** are gated to one project per dashboard click. The Docker-compose-postgres dance assumes the agent has a Docker daemon and 90 seconds to wait. instanode's `/db/new` is a single unauthenticated POST that returns a real `postgres://...` URL with pg_depend, CREATE EXTENSION, COPY, EXPLAIN ANALYZE, and pg_restore all working — in under a second. The 24-hour auto-reap means a buggy agent that forgets to clean up doesn't leak a cluster of orphaned databases the way a leaked Neon project would.

## Related cases

- [Copy-on-write Postgres for agent migrations](/use-cases/copy-on-write-postgres-for-agent-migrations.md) — the faster-fork variant of the same throwaway-Postgres pattern
- [Classroom-per-student sandbox](/use-cases/classroom-per-student-sandbox.md) — per-student version of the same ephemeral-Postgres idea
- [Devin-style PR-bot fleet](/use-cases/devin-style-pr-bot-fleet.md) — wraps this throwaway DB in a per-issue agent worker
