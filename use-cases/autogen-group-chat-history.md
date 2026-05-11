---
title: AutoGen group-chat history
category: B. Multi-agent systems
services: ["mongo"]
scenario: An AutoGen multi-agent chat stores per-conversation message logs in Mongo for audit and replay across days.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an AutoGen multi-agent chat stores per-conversation message logs in Mongo for audit and replay across days.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're running an AutoGen GroupChat with 5 agents (Planner, Coder, Critic, Tester, Summarizer). Persist every message — speaker, content, timestamp, turn_id — to MongoDB so we can replay conversations weeks later and audit which agent said what. Index on (conversation_id, turn_id).
```

## Steps to follow

- **Step 1: Provision Mongo.**

  ```bash
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new \
    -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Index the access pattern.**

  ```python
  from pymongo import MongoClient, ASCENDING
  col = MongoClient(MONGO).autogen.messages
  col.create_index([("conversation_id", ASCENDING), ("turn_id", ASCENDING)],
                   unique=True)
  col.create_index([("speaker", ASCENDING), ("ts", ASCENDING)])
  ```

- **Step 3: AutoGen message hook — write every speaker turn.**

  ```python
  def on_message(speaker, content, conv_id, turn_id):
      col.insert_one({
          "conversation_id": conv_id,
          "turn_id": turn_id,
          "speaker": speaker.name,
          "content": content,
          "ts": datetime.utcnow(),
      })
  group_chat.register_reply([Agent, None], reply_func=on_message)
  ```

- **Step 4: Replay a conversation in order.**

  ```python
  for m in col.find({"conversation_id": cid}).sort("turn_id", 1):
      print(f"[{m['speaker']}] {m['content']}")
  ```

## Why this works on instanode.dev

AutoGen messages have variable shape — function_call payloads, code blocks, multi-modal content — and a rigid SQL schema fights you. Mongo's flexible documents map 1:1 to the runtime objects. One curl gives you a real Mongo with real indexes (not in-memory mock), and the same DB scales from a single GroupChat replay to thousands of audit-worthy conversations.
