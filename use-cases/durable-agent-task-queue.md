---
title: Durable agent task queue
category: B. Multi-agent systems
services: ["nats"]
scenario: A supervisor agent enqueues sub-tasks on a durable queue; failed jobs re-emerge with exponential backoff.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a supervisor agent enqueues sub-tasks on a durable queue; failed jobs re-emerge with exponential backoff.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Supervisor: claim a NATS JetStream on instanode.dev. Enqueue sub-tasks to "tasks.todo" with a retry header. Workers fetch durably; on failure they NAK with delay = 2^attempt seconds. Cap attempts at 5; route to "tasks.dead" beyond.
```

## Steps to follow

- **Step 1: Provision the queue.** Single POST.

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new | jq -r .connection_url)
  ```

- **Step 2: Configure the stream with DLQ.** Two subjects; one is the dead-letter.

  ```python
  await js.add_stream(name="TASKS", subjects=["tasks.todo","tasks.dead"], max_msgs_per_subject=100_000)
  await js.add_consumer("TASKS", ConsumerConfig(durable_name="worker", ack_wait=30, max_deliver=5))
  ```

- **Step 3: Worker pull loop.** NAK with exponential backoff on transient failures.

  ```python
  msgs = await sub.fetch(batch=5, timeout=10)
  for m in msgs:
      try:
          run(json.loads(m.data))
          await m.ack()
      except TransientError:
          delay = 2 ** m.metadata.num_delivered
          await m.nak(delay=delay)
      except PermanentError:
          await js.publish("tasks.dead", m.data); await m.ack()
  ```

- **Step 4: Inspect the dead-letter queue.** Replay or audit.

  ```bash
  nats stream view TASKS --subject tasks.dead --server $NATS_URL
  ```

## Why this works on instanode.dev

JetStream gives true at-least-once with durable consumers — failed messages survive worker crashes and pod restarts. No SQS, no Lambda glue, no AWS account; one curl gives the supervisor a real queue reachable from anywhere. max_deliver caps blast radius for poison messages, and the DLQ subject keeps the main queue clean.

## Related cases

- [Browser job queue with retries](/use-cases/browser-job-queue-with-retries.md) — concrete browser-domain instance of the same retry queue
- [Per-agent dead-letter inspection queue](/use-cases/per-agent-dead-letter-inspection-queue.md) — DLQ side of the same JetStream backoff pattern
- [CrewAI message bus fan-out](/use-cases/crewai-message-bus-fan-out.md) — uses this kind of durable queue under a CrewAI crew
