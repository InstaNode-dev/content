---
title: CRM for one person
category: D. Personal AI
services: ["pg"]
scenario: A personal CRM agent remembers names, birthdays, and last-conversation summaries indexed per contact.
---

## Sample agent prompt

```
Build me a personal CRM. Claim Postgres on instanode.dev. Create contacts(name, birthday, last_seen) and interactions(contact_id, summary, occurred_at). When I say "I just had coffee with Sarah", append an interaction and update last_seen. When I ask "what's new with Sarah", show me her last 3 interactions sorted desc.
```

## Steps to follow

- **Step 1: Provision durable Postgres.** A single POST; the URL persists across machines.

  ```bash
  curl -sX POST https://api.instanode.dev/db/new \
    -H "Content-Type: application/json" -d '{}' | jq
  ```

- **Step 2: Create the schema.** Simple, two tables, no ORM needed.

  ```sql
  CREATE TABLE contacts (
    id bigserial PRIMARY KEY,
    name text UNIQUE NOT NULL,
    birthday date,
    last_seen timestamptz
  );
  CREATE TABLE interactions (
    id bigserial PRIMARY KEY,
    contact_id bigint REFERENCES contacts(id),
    summary text,
    occurred_at timestamptz DEFAULT now()
  );
  ```

- **Step 3: Log a coffee.** Upsert contact, then insert interaction.

  ```sql
  WITH c AS (
    INSERT INTO contacts(name) VALUES ('Sarah')
    ON CONFLICT (name) DO UPDATE SET last_seen = now() RETURNING id
  )
  INSERT INTO interactions(contact_id, summary)
  SELECT id, 'coffee at Sightglass, talked about her new role' FROM c;
  ```

- **Step 4: Recall on demand.** Three rows, ordered.

  ```sql
  SELECT summary, occurred_at FROM interactions
  WHERE contact_id = (SELECT id FROM contacts WHERE name='Sarah')
  ORDER BY occurred_at DESC LIMIT 3;
  ```

## Why this works on instanode.dev

Claim the token once and the same Postgres URL works from your laptop, phone (via Termux), and a Raspberry Pi sitting at home — no need to expose a personal DB to the internet yourself. AES-256-GCM at rest plus a unique random password means a leaked URL is recoverable: rotate via the API, and your old URL stops working without losing data.
