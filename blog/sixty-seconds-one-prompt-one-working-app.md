---
title: Sixty seconds, one prompt, one working app
date: 2026-05-12
author: instanode.dev
excerpt: Devika asked her agent for "a tiny expense tracker I can hit from my phone." Four curls later, a FastAPI app was answering on the public internet. Here is the exact session, copy-pasteable.
---

# Sixty seconds, one prompt, one working app

Devika has been writing down her spending in a Notes app for three years.
It is, by any reasonable measure, a mess. On Sunday evening she opened
Claude Code and typed one sentence:

```
Build me a tiny expense tracker. Postgres backing store, FastAPI on top,
deploy it somewhere I can hit from my phone. Use instanode.dev.
```

Four curls and ninety-three seconds later there was a working
`https://expense-<your-id>.deployment.instanode.dev` answering HTTP. She added
three transactions from her phone in line at the coffee shop the next
morning. This post is the exact transcript of that session — every
command, every response shape, nothing skipped.

## The prompt is the plan

The agent read [/llms.txt](https://instanode.dev/llms.txt) before doing
anything. That single page documents every endpoint, every response
field, and the anonymous-tier limits. The agent now knows: provision is
a POST, the response includes a `connection_url`, the deploy endpoint
takes a multipart tarball, and an `upgrade_jwt` falls out of the first
provision call so the deploy step can authenticate.

No SDK, no MCP setup, no API key. The agent's only "integration" is the
shape of the JSON it gets back.

## Step 1: Postgres in 950 ms

```bash
curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"sixty-seconds-one-prompt-one-worki-db"}' | tee pg.json
```

Response (trimmed):

```json
{
  "ok": true,
  "token": "tok_b3f2...",
  "connection_url": "postgres://u_b3f2:...@pg-shared-3.instanode.dev:5432/db_b3f2",
  "tier": "anonymous",
  "limits": {"storage_mb": 10, "connections": 2},
  "upgrade_jwt": "eyJhbGc..."
}
```

The agent stashes `connection_url` for the app and `upgrade_jwt` for the
deploy step. The Postgres is a real dedicated database on the shared
cluster — `psql` works against it from the agent's shell instantly.

## Step 2: Three tables, one migration

The agent writes the schema. No ORM, no migration tool, just SQL piped
into `psql`.

```bash
psql "$(jq -r .connection_url pg.json)" <<'SQL'
CREATE TABLE categories (
  id bigserial PRIMARY KEY,
  name text UNIQUE NOT NULL
);

CREATE TABLE transactions (
  id bigserial PRIMARY KEY,
  amount_cents int NOT NULL,
  category_id bigint REFERENCES categories(id),
  note text,
  occurred_at timestamptz DEFAULT now()
);

CREATE TABLE monthly_budgets (
  id bigserial PRIMARY KEY,
  category_id bigint REFERENCES categories(id),
  month date NOT NULL,
  limit_cents int NOT NULL,
  UNIQUE (category_id, month)
);

INSERT INTO categories(name) VALUES ('groceries'), ('coffee'), ('transport');
SQL
```

Three tables, a sensible foreign-key structure, three seed categories.
The agent didn't have to ask which database it was talking to — the URL
told it.

## Step 3: A small FastAPI app

The whole app fits in one file. The agent writes `app.py` directly:

```python
import os, psycopg
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()
DB = os.environ["DATABASE_URL"]

class Tx(BaseModel):
    amount_cents: int
    category: str
    note: str | None = None

@app.get("/healthz")
def healthz(): return {"ok": True}

@app.post("/tx")
def add_tx(t: Tx):
    with psycopg.connect(DB) as conn, conn.cursor() as cur:
        cur.execute(
            "INSERT INTO transactions(amount_cents, category_id, note) "
            "SELECT %s, id, %s FROM categories WHERE name=%s "
            "RETURNING id, occurred_at",
            (t.amount_cents, t.note, t.category),
        )
        row = cur.fetchone()
    return {"id": row[0], "at": row[1].isoformat()}

@app.get("/summary")
def summary():
    with psycopg.connect(DB) as conn, conn.cursor() as cur:
        cur.execute("""
          SELECT c.name, SUM(t.amount_cents)::int
          FROM transactions t JOIN categories c ON c.id = t.category_id
          WHERE t.occurred_at >= date_trunc('month', now())
          GROUP BY c.name ORDER BY 2 DESC
        """)
        return [{"category": r[0], "spent_cents": r[1]} for r in cur.fetchall()]
```

Plus a two-line `requirements.txt` (`fastapi`, `psycopg[binary]`) and a
tiny `Dockerfile` that pip-installs those and runs
`uvicorn app:app --host 0.0.0.0 --port 8080`. The agent generates all of
this in the working directory.

## Step 4: Deploy

```bash
tar -czf app.tar.gz Dockerfile requirements.txt app.py

curl -X POST https://api.instanode.dev/deploy/new \
  -H "Authorization: Bearer $(jq -r .upgrade_jwt pg.json)" \
  -F "tarball=@app.tar.gz" \
  -F "name=expense" \
  -F "port=8080" \
  -F "env_vars={\"DATABASE_URL\":\"$(jq -r .connection_url pg.json)\"}"
```

The build runs in-cluster via kaniko — about 90 seconds the first time
because the Python base image has to download into the build context.
The response:

```json
{
  "ok": true,
  "token": "tok_dep_9b2c...",
  "app_url": "https://expense-<your-id>.deployment.instanode.dev",
  "build_status": "running",
  "tier": "anonymous"
}
```

The build status flips to `succeeded` about a minute later; the
TLS-terminated public URL is live with a Let's Encrypt cert the platform
issued on first hit.

## Step 5: Use the app

Copy the `app_url` the platform returned in Step 4 (it'll have a
different random suffix from the placeholder below — whatever the API
hands back is yours).

```bash
APP=$(jq -r .app_url deploy.json)
# or paste the URL: APP=https://expense-<your-id>.deployment.instanode.dev

curl "$APP/healthz"
# expected: {"ok":true}

curl -X POST "$APP/tx" -H "content-type: application/json" \
  -d '{"amount_cents": 480, "category": "coffee", "note": "oat latte"}'
# expected: {"id":1,"at":"2026-..."}

curl "$APP/summary"
# expected: [{"category":"coffee","spent_cents":480}]
```

That's the app. Devika is now logging expenses from her phone.

## Step 6 (optional): Claim it

Anonymous resources expire in 24 hours. If Devika decides this is worth
keeping she follows the URL printed in the `/db/new` response:

```
https://api.instanode.dev/start?t=eyJhbGc...
```

That 302s into the dashboard with the JWT pre-filled. One click claims
the Postgres, the deploy, and binds both to her account. After that
they live indefinitely at the [hobby tier](https://instanode.dev/pricing)
($9/mo for 1 GB Postgres + one small deploy app — see the
[anonymous-as-trial](/blog/why-anonymous-is-the-trial) post for the
reasoning).

If your fingerprint has previously claimed and the free resource has
since expired, `/db/new` will return a 402 asking you to claim again
first at https://instanode.dev/claim — one-time email gate, no card,
30 seconds. Subsequent provisions then go through as shown above.

## Why this fits in 60 seconds

Every alternative ships its own ceremony. RDS wants a VPC, a subnet
group, an inbound rule, and a parameter group. Heroku Postgres wants a
verified email and a credit card. Supabase wants a project. Fly Machines
wants `flyctl auth login` and a region picker. Each of those is fine
once. Each of them assumes Devika is here for the platform, not for the
expense tracker.

The instanode endpoints don't introduce themselves. The agent typed what
it was going to type anyway — `curl -X POST /db/new`, `curl -X POST
/deploy/new` — and the platform replied with the actual thing. The
build was the work. The infra was a side effect.

## Cross-links

- [Side-project container deploy](/use-cases/side-project-container-deploy) — the deploy primitive in isolation, with the same multipart shape
- [One-afternoon MVP backend](/use-cases/one-afternoon-mvp-backend) — the same Postgres + deploy bundle for a paid product instead of a personal tool
- [CRM for one person](/use-cases/crm-for-one-person) — single-user Postgres pattern that pairs naturally with this tracker
- [llms.txt](https://instanode.dev/llms.txt) — the one-page API reference every agent reads first

The curl works right now. No signup.
