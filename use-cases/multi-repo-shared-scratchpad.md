---
title: Multi-repo shared scratchpad
category: A. AI coding agents
services: ["mongo"]
scenario: Multiple agents working across separate repos coordinate via a shared scratchpad document store keyed by feature branch.
---

## Sample agent prompt

```
Spin up a shared Mongo scratchpad so the four coding agents working across our monorepo + two satellite repos can leave notes for each other, keyed by feature branch. The schema is flexible (each note is a JSON doc). Provision Mongo and return the URL plus an example insert/query.
```

## Steps to follow

- **Step 1: Provision Mongo.**

  ```bash
  curl -s -X POST https://api.instanode.dev/nosql/new | jq -r .connection_url
  ```

- **Step 2: Connect and pick a collection per branch.**

  ```python
  from pymongo import MongoClient
  m = MongoClient(os.environ["MONGO_URL"])["scratch"]["notes"]
  m.create_index([("branch", 1), ("ts", -1)])
  ```

- **Step 3: Agent leaves a note.**

  ```python
  m.insert_one({
      "branch": "feat/billing-rework",
      "agent": "frontend-1",
      "kind": "blocker",
      "ref": "packages/web/src/Checkout.tsx:142",
      "note": "Stripe element re-renders on form change; need useMemo",
      "ts": datetime.utcnow()
  })
  ```

- **Step 4: Another agent reads the latest on that branch before starting.**

  ```python
  for n in m.find({"branch": "feat/billing-rework"}).sort("ts", -1).limit(20):
      print(n["agent"], n["note"])
  ```

- **Step 5: GC stale branch notes weekly.**

  ```javascript
  db.notes.deleteMany({ ts: { $lt: new Date(Date.now() - 14*864e5) } })
  ```

## Why this works on instanode.dev

Mongo's schemaless collections fit agent notes that drift in shape across repos, and the branch-keyed index makes cross-agent reads cheap. One curl produces a working URL — no Atlas account, no IP allowlist.
