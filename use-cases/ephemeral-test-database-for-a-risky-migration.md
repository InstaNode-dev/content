---
title: Ephemeral test database for a risky migration
category: A. AI coding agents
services: ["pg"]
scenario: A coding agent provisions a throwaway Postgres in under a second, dry-runs a destructive migration, inspects the result, then discards the database — no production blast radius.
---

## Sample agent prompt

```
I have a migration that does ALTER TABLE orders DROP COLUMN customer_email; it terrifies me. Claim a Postgres on instanode.dev, restore last night's dump into it, run the migration in a transaction, check that no app code references the column, and ROLLBACK. Print whether it's safe.
```

## Steps to follow

- **Step 1: Claim a throwaway database.** Ephemeral, 24h auto-reap.

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  ```

- **Step 2: Seed with the prod snapshot.** Use pg_restore from your nightly backup.

  ```bash
  pg_restore --no-owner --no-acl -d $PG /backups/prod-2026-05-10.dump
  ```

- **Step 3: Dry-run the dangerous migration.** Transaction-wrapped so you can inspect.

  ```sql
  BEGIN;
  ALTER TABLE orders DROP COLUMN customer_email;
  -- check for downstream issues
  SELECT n.nspname, c.relname FROM pg_depend d
    JOIN pg_class c ON d.refobjid = c.oid
    JOIN pg_namespace n ON c.relnamespace = n.oid
   WHERE d.refobjsubid = (SELECT attnum FROM pg_attribute WHERE attrelid = 'orders'::regclass AND attname = 'customer_email');
  ROLLBACK;
  ```

- **Step 4: Greg the codebase too.** Catch app-layer references the DB doesn't know about.

  ```bash
  rg "customer_email" src/ tests/ migrations/
  ```

## Why this works on instanode.dev

A real Postgres with CREATE EXTENSION, REINDEX, ALTER, COPY, EXPLAIN — not a SQLite stand-in or a Docker image you have to build. Sub-second provisioning means the agent doesn't wait around; the migration is tested before the coffee is poured. Auto-expiry means a forgotten test DB never accrues cost.
