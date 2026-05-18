---
title: Form-fill state machine
category: E. Browser & automation agents
services: ["mongo"]
scenario: A long-running form-completion agent persists field-by-field progress so a captcha pause doesn't lose 30 minutes of work.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a long-running form-completion agent persists field-by-field progress so a captcha pause doesn't lose 30 minutes of work.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're filling a 200-field government form via browser automation. Claim a MongoDB on instanode.dev. After each field, upsert the entire form state to mongo.form_runs. On a captcha pause or browser crash, resume from the latest state doc and skip already-filled fields.
```

## Steps to follow

- **Step 1: Provision Mongo.** One POST.

  ```bash
  curl -sX POST https://api.instanode.dev/nosql/new -H 'Content-Type: application/json' -d '{"name":"form-fill-state-machine-mongo"}' | jq -r .connection_url
  ```

- **Step 2: Define the document shape.** Nested object: section → field → value + filled_at.

  ```javascript
  db.form_runs.createIndex({ run_id: 1 }, { unique: true });
  // shape: {run_id, fields: {field1: {value, filled_at}, ...}, status, ts}
  ```

- **Step 3: After each field, persist state.** Atomic $set.

  ```python
  mongo.form_runs.update_one(
      {"run_id": run_id},
      {"$set": {f"fields.{field}": {"value": value, "filled_at": time.time()}, "ts": time.time()}},
      upsert=True
  )
  ```

- **Step 4: On resume, skip filled fields.** Crash-safe.

  ```python
  state = mongo.form_runs.find_one({"run_id": run_id})
  for f in form.fields:
      if f.name in state["fields"]: continue
      browser.type(f.selector, derive_value(f))
  ```

## Why this works on instanode.dev

Mongo's per-field atomic `$set` is exactly right for incremental state-machine progress — no need to serialize a giant JSON blob on every keystroke. The connection URL is reachable from any browser-automation runner (Browserbase, your own Playwright pods, anything), so a session that started on a laptop can resume in headless cloud. AES-256 encryption at rest matters here because government forms carry PII.

## Related cases

- [Browser-session cookie store](/use-cases/browser-session-cookie-store.md) — the auth-state cache that lets the form session resume
- [Accessibility-tree selector cache](/use-cases/accessibility-tree-selector-cache.md) — the selector layer this state machine fills against
- [LangGraph state checkpoints](/use-cases/langgraph-state-checkpoints.md) — framework-level version of the same crash-resumable pattern
