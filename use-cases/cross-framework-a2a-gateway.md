---
title: Cross-framework A2A gateway
category: B. Multi-agent systems
services: ["nats", "pg"]
scenario: A gateway service translates A2A-protocol messages between a LangGraph crew and a CrewAI crew over pub/sub.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a gateway service translates A2A-protocol messages between a LangGraph crew and a CrewAI crew over pub/sub.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build an A2A gateway. Claim NATS + Postgres on instanode.dev. NATS carries the message bus; Postgres logs every translated message with source_framework, target_framework, and payload. The gateway service converts LangGraph state-graph messages to CrewAI task messages and vice versa.
```

## Steps to follow

- **Step 1: Provision the bus + log.** Two curls, same anonymous token.

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new -H 'Content-Type: application/json' -d '{"name":"cross-framework-a2a-gateway-queue"}' | jq -r .connection_url)
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"cross-framework-a2a-gateway-db"}' | jq -r .connection_url)
  ```

- **Step 2: Create the audit schema.** Every translation is recorded.

  ```sql
  CREATE TABLE a2a_translations (
    id bigserial PRIMARY KEY,
    received_at timestamptz DEFAULT now(),
    src_framework text, src_msg jsonb,
    dst_framework text, dst_msg jsonb
  );
  ```

- **Step 3: Gateway subscribes to both inbound subjects.** Translates and republishes.

  ```python
  async def from_langgraph(msg):
      lg = json.loads(msg.data)
      crew_msg = {"task": lg["next_node"], "context": lg["state"]}
      await js.publish("a2a.crewai.in", json.dumps(crew_msg).encode())
      cur.execute("INSERT INTO a2a_translations (src_framework,src_msg,dst_framework,dst_msg) VALUES ('langgraph',%s,'crewai',%s)",
                  (json.dumps(lg), json.dumps(crew_msg)))
  ```

- **Step 4: Sanity check both crews.** Confirm round-trip.

  ```sql
  SELECT src_framework, dst_framework, COUNT(*) FROM a2a_translations
  WHERE received_at > now() - interval '5 min' GROUP BY 1,2;
  ```

## Why this works on instanode.dev

A gateway is exactly the kind of "stateless service connecting two stateful systems" that benefits most from one-curl infra. You don't want to manage NATS clustering or Postgres backups when the gateway itself is the interesting part. Both services are reachable from any cloud, so the LangGraph crew on AWS and the CrewAI crew on Modal both connect with no VPC peering.

## Related cases

- [A2A agent-card registry](/use-cases/a2a-agent-card-registry) — the discovery layer this gateway routes to
- [Cross-agent shared inbox](/use-cases/cross-agent-shared-inbox) — the durable-subject primitive the gateway sits on
- [OpenAI Agents SDK handoff mesh](/use-cases/openai-agents-sdk-handoff-mesh) — one of the framework dialects this gateway bridges
