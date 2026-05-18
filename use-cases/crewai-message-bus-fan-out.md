---
title: CrewAI message bus fan-out
category: B. Multi-agent systems
services: ["nats"]
scenario: A crew of planner / retriever / critic agents publishes tasks on NATS subjects so any worker pod can pull and reply.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a crew of planner / retriever / critic agents publishes tasks on NATS subjects so any worker pod can pull and reply.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Wire a CrewAI crew with a NATS message bus from instanode.dev. Planner publishes to tasks.plan, retriever subscribes and publishes to tasks.retrieve, critic subscribes to tasks.retrieve and publishes to tasks.critique. Use durable consumers so a crashed worker resumes from where it left off.
```

## Steps to follow

- **Step 1: Claim a NATS JetStream URL.** One POST returns a server URL + token-scoped credentials.

  ```bash
  NATS_URL=$(curl -sX POST https://api.instanode.dev/queue/new -H 'Content-Type: application/json' -d '{"name":"crewai-message-bus-fan-out-queue"}' | jq -r .connection_url)
  ```

- **Step 2: Create the stream + subjects.** JetStream gives durability; standard NATS would be fire-and-forget.

  ```python
  import nats
  nc = await nats.connect(NATS_URL)
  js = nc.jetstream()
  await js.add_stream(name="CREW", subjects=["tasks.>"])
  ```

- **Step 3: Wire CrewAI agents as pub/sub workers.** Each agent is its own durable consumer.

  ```python
  async def planner(task):
      plan = planner_agent.kickoff(inputs={"task": task})
      await js.publish("tasks.retrieve", plan.encode())

  await js.subscribe("tasks.plan", durable="planner", cb=planner)
  ```

- **Step 4: Verify with a published task.** Watch all three subjects fire.

  ```bash
  nats pub tasks.plan "summarize Q3 board deck" --server $NATS_URL
  nats sub "tasks.>" --server $NATS_URL
  ```

## Why this works on instanode.dev

NATS JetStream is a single-curl provision, not a cluster you operate. The server is reachable from anywhere — laptops, k8s pods, Vercel functions — so a heterogeneous crew running across different runtimes shares the same bus. Token-scoped credentials mean each crew gets its own isolated subject namespace; there's no risk of crews stomping on each other.

## Related cases

- [LangGraph fan-out research agents](/use-cases/langgraph-fan-out-research-agents.md) — framework-equivalent fan-out pattern with a Postgres reducer
- [Cross-framework A2A gateway](/use-cases/cross-framework-a2a-gateway.md) — translates CrewAI subjects into other frameworks' protocols
- [Durable agent task queue](/use-cases/durable-agent-task-queue.md) — the durability-with-retry primitive these CrewAI subjects need
