---
title: AP2 mandate audit trail
category: G. Internet-of-AI
services: ["pg", "storage"]
scenario: An agentic-commerce gateway stores signed user mandates ("buy X up to $Y") for later dispute resolution.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an agentic-commerce gateway stores signed user mandates ("buy X up to $Y") for later dispute resolution.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + S3-compatible storage) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You operate an agentic-commerce gateway. Every user mandate ("buy this SKU up to $300, valid until Friday") arrives signed. Store the canonical mandate in Postgres for indexed queries (by user, by expiry) and the full signed JWS blob in S3-compatible storage for dispute resolution. On every purchase, link the txn back to the mandate row.
```

## Steps to follow

- **Step 1: Provision DB + bucket.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"ap2-mandate-audit-trail-db"}' -H "Authorization: Bearer $T" | jq -r .connection_url)
  curl -sX POST https://api.instanode.dev/storage/new -H 'Content-Type: application/json' -d '{"name":"ap2-mandate-audit-trail-storage"}' -H "Authorization: Bearer $T" > s3.json
  ```

- **Step 2: Mandate metadata table.**

  ```sql
  CREATE TABLE mandates (
    mandate_id uuid PRIMARY KEY,
    user_id text NOT NULL,
    sku text,
    max_amount_cents int NOT NULL,
    expires_at timestamptz NOT NULL,
    signed_blob_key text NOT NULL,
    status text DEFAULT 'active',
    created_at timestamptz DEFAULT now()
  );
  CREATE INDEX idx_user_active ON mandates (user_id, status, expires_at);
  ```

- **Step 3: Store the JWS in S3-compatible storage, the metadata in PG.**

  ```python
  key = f"mandates/{user_id}/{mandate_id}.jws"
  s3.put_object(Bucket="instant-shared", Key=key, Body=jws_blob,
                ContentType="application/jose")
  pg.execute("""INSERT INTO mandates (mandate_id, user_id, sku, max_amount_cents,
                expires_at, signed_blob_key)
                VALUES (%s,%s,%s,%s,%s,%s)""",
             (mid, uid, sku, cents, exp, key))
  ```

- **Step 4: Dispute resolution — fetch the original signed payload.**

  ```python
  row = pg.fetchone("SELECT signed_blob_key FROM mandates WHERE mandate_id=%s", (mid,))
  blob = s3.get_object(Bucket="instant-shared", Key=row["signed_blob_key"])["Body"].read()
  verify_jws(blob, public_key)  # proves user actually signed it
  ```

## Why this works on instanode.dev

AP2 mandates need both fast metadata queries (Postgres) and tamper-proof raw-blob retention (object storage). Splitting them keeps the hot path indexed and the cold blobs cheap. Two curls, both real services, both encrypted at rest — exactly the shape a compliance auditor expects to see, without an AWS-Organizations setup project.

## Related cases

- [AP2 mandate broker](/use-cases/ap2-mandate-broker.md) — the live-broker counterpart to this audit-only store
- [Agent-marketplace escrow](/use-cases/agent-marketplace-escrow.md) — another signed-commitment ledger for agent commerce
- [x402 micropayment ledger](/use-cases/x402-micropayment-ledger.md) — the micropayment counterpart that this mandate authorizes
