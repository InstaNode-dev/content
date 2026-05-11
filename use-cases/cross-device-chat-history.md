---
title: Cross-device chat history
category: D. Personal AI
services: ["mongo"]
scenario: A personal assistant stores conversation history in Mongo so the same agent picks up context on phone, laptop, and watch.
---

## Sample agent prompt

```
I want my assistant's chat history to follow me from phone to laptop to watch. Claim a MongoDB on instanode.dev. On every message, upsert into conversations(user_id, ts, role, content). When the assistant boots on a new device, replay the last 50 messages as context.
```

## Steps to follow

- **Step 1: Claim Mongo.** Single POST; URL persists indefinitely on hobby tier.

  ```bash
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new | jq -r .connection_url)
  ```

- **Step 2: Define the collection + indexes.** Compound index for fast per-user scroll.

  ```javascript
  db.messages.createIndex({ user_id: 1, ts: -1 });
  db.messages.createIndex({ user_id: 1, role: 1, ts: -1 });
  ```

- **Step 3: Append every message from any device.** All devices share the same Mongo URL.

  ```python
  mongo.messages.insert_one({
      "user_id": "manas",
      "ts": time.time(),
      "role": role,
      "content": text,
      "device": platform.node()
  })
  ```

- **Step 4: Restore context on boot.** Read last N, feed into the system prompt.

  ```python
  history = list(mongo.messages.find({"user_id": "manas"}).sort("ts", -1).limit(50))
  history.reverse()
  context = [{"role": m["role"], "content": m["content"]} for m in history]
  ```

## Why this works on instanode.dev

The Mongo URL is public-internet reachable, so a watch on cell data or a laptop on hotel Wi-Fi both connect with the same string. Mongo's flexible schema means you can evolve message format (add tool_calls, attachments, citations) without an ALTER. AES-256-GCM means a leaked URL on a device is recoverable: rotate via the API and the old credentials die.
