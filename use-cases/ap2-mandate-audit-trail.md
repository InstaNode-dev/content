---
title: AP2 mandate audit trail
category: G. Internet-of-AI
services: ["pg", "minio"]
scenario: An agentic-commerce gateway stores signed user mandates ("buy X up to $Y") for later dispute resolution.
---

## Sample agent prompt

```
You operate an agentic-commerce gateway. Every user mandate ("buy this SKU up to $300, valid until Friday") arrives signed. Store the canonical mandate in Postgres for indexed queries (by user, by expiry) and the full signed JWS blob in MinIO for dispute resolution. On every purchase, link the txn back to the mandate row.
```

## Steps to follow

- **Step 1: Provision DB + bucket.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "Authorization: Bearer $T" | jq -r .connection_url)
  curl -sX POST https://api.instanode.dev/storage/new -H "Authorization: Bearer $T" > s3.json
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

- **Step 3: Store the JWS in MinIO, the metadata in PG.**

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
