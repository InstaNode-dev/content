---
title: Cross-agent shared inbox
category: G. Internet-of-AI
services: ["nats"]
scenario: Two agents negotiate over a shared subject; messages persist in JetStream so neither needs to be online simultaneously.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: two agents negotiate over a shared subject; messages persist in JetStream so neither needs to be online simultaneously.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Two agents need to negotiate asynchronously. Claim a NATS JetStream on instanode.dev and create a stream "negotiation" with subjects negotiation.A and negotiation.B. Each agent publishes to its own subject and subscribes durably to the other's. Either agent can be offline for hours; messages persist.
```

## Steps to follow

- **Step 1: Provision durable NATS.** Returns a connection URL and credentials.

  ```bash
  curl -sX POST https://api.instanode.dev/queue/new \
    -H "Content-Type: application/json" \
    -d '{"stream":"negotiation","subjects":["negotiation.A","negotiation.B"]}' | jq
  ```

- **Step 2: Agent A subscribes durably to B's subject.** Durable = position survives reconnect.

  ```python
  await js.pull_subscribe("negotiation.B", durable="agent_A")
  while True:
      msgs = await sub.fetch(1, timeout=60)
      for m in msgs:
          await handle(m.data); await m.ack()
  ```

- **Step 3: Agent A publishes a counter-offer.** Even if B is offline, JetStream holds it.

  ```python
  await js.publish("negotiation.A", json.dumps({"price": 1200, "terms": "net-30"}).encode())
  ```

- **Step 4: Inspect the stream.** Replay or audit the whole negotiation later.

  ```bash
  nats stream view negotiation --server $NATS_URL
  ```

## Why this works on instanode.dev

JetStream is a real durable stream — messages survive broker restarts, agent crashes, and overnight disconnects — provisioned in one curl. Neither agent needs to host an SMTP server, run a poller against email, or coordinate timezones; the inbox is the broker. For the Internet-of-AI case where agents speak across vendors, instanode's URL is reachable from anywhere and authenticated by token, no peering required.

## Related cases

- [Cross-framework A2A gateway](/use-cases/cross-framework-a2a-gateway.md) — promotes the inbox into a protocol-translating bus
- [Durable agent task queue](/use-cases/durable-agent-task-queue.md) — same JetStream durability primitive for jobs instead of conversations
- [A2A agent-card registry](/use-cases/a2a-agent-card-registry.md) — directory that tells agents which inbox to negotiate on
