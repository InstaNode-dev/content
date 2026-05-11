---
title: Personal-injury demand letters
category: C. Vertical AI apps
services: ["pg", "webhook"]
scenario: A legal agent drafts demand letters from case files, stores prior versions, and emits webhooks to the firm's case-management system.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a legal agent drafts demand letters from case files, stores prior versions, and emits webhooks to the firm's case-management system.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + webhook receiver) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a legal agent that drafts personal-injury demand letters from intake files. Each draft version persists to Postgres with full history, and on send we emit a webhook to Clio (the firm's case-management system) with the matter_id and PDF URL. Provision Postgres and a webhook sender endpoint.
```

## Steps to follow

- **Step 1: Provision Postgres and a webhook (we'll use it to forward to Clio).**

  ```bash
  curl -s -X POST https://api.instanode.dev/db/new
  curl -s -X POST https://api.instanode.dev/webhook/new
  ```

- **Step 2: Letter version schema.**

  ```sql
  CREATE TABLE letters (
    matter_id text NOT NULL,
    version int NOT NULL,
    draft text NOT NULL,
    author text,
    created_at timestamptz DEFAULT now(),
    PRIMARY KEY (matter_id, version)
  );
  CREATE INDEX ON letters (matter_id, version DESC);
  ```

- **Step 3: Drafting flow inside the agent.**

  ```python
  prior = pg.fetchrow("SELECT draft FROM letters WHERE matter_id=%s ORDER BY version DESC LIMIT 1", mid)
  intake = load_intake(mid)
  new_draft = llm.draft(intake=intake, prior=prior["draft"] if prior else None)
  pg.execute("INSERT INTO letters VALUES (%s,(SELECT coalesce(max(version),0)+1 FROM letters WHERE matter_id=%s),%s,%s)",
             mid, mid, new_draft, "agent-v3")
  ```

- **Step 4: On final send, POST to Clio via webhook.**

  ```python
  requests.post(CLIO_WEBHOOK, json={
      "matter_id": mid,
      "pdf_url": uploaded_pdf,
      "sent_at": datetime.utcnow().isoformat()
  })
  ```

- **Step 5: Audit query — every version of a matter.**

  ```sql
  SELECT version, author, created_at FROM letters WHERE matter_id=$1 ORDER BY version;
  ```

## Why this works on instanode.dev

Letters need a complete edit history for malpractice defense; Postgres versioning gives it cheaply. The webhook receiver lets you decouple from Clio's API quirks — buffer first, retry second.

## Related cases

- [Clinical-scribe note storage](/use-cases/clinical-scribe-note-storage.md) — another vertical-AI workflow with versioned per-case docs
- [Contract redline cache](/use-cases/contract-redline-cache.md) — adjacent legal-AI workflow that caches clause embeddings
- [AML transaction monitor](/use-cases/aml-transaction-monitor.md) — compliance-flavored sibling with full auditable reasoning trace
