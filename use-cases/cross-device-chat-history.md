---
title: Cross-device chat history
category: D. Personal AI
services: ["mongo"]
scenario: A personal assistant stores conversation history in Mongo so the same agent picks up context on phone, laptop, and watch.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a personal assistant stores conversation history in Mongo so the same agent picks up context on phone, laptop, and watch.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

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

## Related cases

- [AutoGen group-chat history](/use-cases/autogen-group-chat-history.md) — the multi-agent equivalent of this per-user Mongo log
- [Daily-journal episodic memory](/use-cases/daily-journal-episodic-memory.md) — adjacent personal-AI Postgres+Redis recall pattern
- [Conversation transcript archive](/use-cases/conversation-transcript-archive.md) — the team-wide append-only sibling of cross-device history
