---
title: LangGraph fan-out research agents
category: J. Agent swarms & fan-out
services: ["nats", "pg"]
scenario: A planner agent splits a research question into 12 sub-queries and dispatches them to parallel LangGraph workers that each hit different sources, returning JSON to a NATS subject the reducer subscribes to.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a planner agent splits a research question into 12 sub-queries and dispatches them to parallel LangGraph workers that each hit different sources, returning JSON to a NATS subject the reducer subscribes to.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
I need to write a literature review on "GLP-1 receptor agonists and cardiovascular outcomes in non-diabetic patients" by tomorrow morning. Build me a LangGraph planner that decomposes the question into 12 sub-queries (mechanism, RCT evidence, observational data, MACE endpoints, side effects, cost-effectiveness, guideline positions, etc.), fans them out as NATS JetStream messages on `research.work.<run_id>`, and runs 12 parallel workers that each query PubMed + Semantic Scholar + a web search and write their raw findings (with citations) to a Postgres `findings` table. The reducer subscribes to `research.done.<run_id>`, collects all 12 results, and synthesizes a 1500-word draft with inline citations. Provision NATS + Postgres on instanode.dev and print the URLs.
```

## Steps to follow

We'll thread one query — the GLP-1 literature review above — through every step. By the end you'll have 12 worker results in Postgres and a synthesized draft in stdout.

- **Step 1: Provision NATS JetStream + Postgres in two POSTs.** The same claimed token owns both; workers receive both URLs as env.

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new \
           -H 'Content-Type: application/json' \
           -d '{"name":"langgraph-fan-out-research-agents-queue","stream":"research","subjects":["research.work.*","research.done.*"]}' \
         | jq -r .nats_url)
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"langgraph-fan-out-research-agents-db"}' | jq -r .connection_url)
  printf "NATS_URL=%s\nPG_URL=%s\n" "$NATS" "$PG" > .env
  ```

- **Step 2: Persistence schema for the findings.** A composite primary key on `(run_id, sub_query)` makes idempotent re-deliveries safe.

  ```sql
  CREATE TABLE findings (
    run_id      uuid NOT NULL,
    sub_query   text NOT NULL,
    sources     jsonb NOT NULL,
    summary     text NOT NULL,
    received_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (run_id, sub_query)
  );
  CREATE INDEX findings_by_run ON findings(run_id);
  ```

- **Step 3: The planner LangGraph node.** Decomposes the question with the LLM, publishes 12 messages onto `research.work.<run_id>`. Each message carries the sub-query and the run_id so workers and the reducer share the same correlation key.

  ```python
  # graph/planner.py
  import json, uuid, nats
  from nats.js import JetStreamContext

  PROMPT = """Decompose this medical literature question into exactly 12
  non-overlapping sub-queries that together cover the topic. Return JSON
  list of strings.

  Question: {q}"""

  async def planner(state):
      run_id = str(uuid.uuid4())
      sub_queries = await llm_json(PROMPT.format(q=state["question"]))
      assert len(sub_queries) == 12, f"expected 12, got {len(sub_queries)}"
      nc = await nats.connect(os.environ["NATS_URL"])
      js: JetStreamContext = nc.jetstream()
      for sq in sub_queries:
          await js.publish(
              f"research.work.{run_id}",
              json.dumps({"q": sq, "run": run_id}).encode(),
          )
      return {**state, "run_id": run_id, "sub_queries": sub_queries}
  ```

- **Step 4: Workers — 12 of them, durably subscribed.** A durable consumer means a crashed worker's message redelivers to a sibling; JetStream's per-subject filter keeps concurrent research runs isolated.

  ```python
  # graph/worker.py
  async def worker():
      nc = await nats.connect(os.environ["NATS_URL"])
      js = nc.jetstream()
      sub = await js.subscribe("research.work.*", durable="workers", manual_ack=True)
      async for msg in sub.messages:
          payload = json.loads(msg.data)
          run_id, q = payload["run"], payload["q"]
          sources = await fetch_sources(q)            # pubmed + semscholar + web
          summary = await llm_summarize(q, sources)
          with psycopg.connect(os.environ["PG_URL"]) as cn:
              cn.execute(
                  "INSERT INTO findings (run_id, sub_query, sources, summary) "
                  "VALUES (%s, %s, %s, %s) "
                  "ON CONFLICT (run_id, sub_query) DO NOTHING",
                  (run_id, q, json.dumps(sources), summary),
              )
          await js.publish(f"research.done.{run_id}",
                           json.dumps({"q": q, "summary": summary}).encode())
          await msg.ack()
  ```

- **Step 5: Reducer waits for 12 done-messages, scoped to this run.** The per-subject filter (`research.done.<run_id>`) is what lets ten concurrent research runs share one JetStream without cross-talk.

  ```python
  async def reducer(state):
      run_id = state["run_id"]
      nc = await nats.connect(os.environ["NATS_URL"])
      js = nc.jetstream()
      sub = await js.subscribe(f"research.done.{run_id}")
      collected = []
      for _ in range(12):
          msg = await sub.next_msg(timeout=120)
          collected.append(json.loads(msg.data))
      draft = await llm_synthesize(state["question"], collected)
      return {**state, "draft": draft}
  ```

- **Step 6: Run end-to-end and verify.** Twelve rows land in Postgres, the reducer emits the synthesized draft.

  ```bash
  python -m graph.run --question "GLP-1 receptor agonists and cardiovascular outcomes in non-diabetic patients"
  psql "$PG_URL" -c "SELECT count(*) FROM findings WHERE run_id = (SELECT run_id FROM findings ORDER BY received_at DESC LIMIT 1);"
  #  count
  # -------
  #     12
  ```

## Why this works on instanode.dev

Fan-out research swarms have historically been gated by two costs that look small individually and ruin the pattern in aggregate. The **broker problem**: NATS, RabbitMQ, or Kafka all want a config file, a TLS cert, and a port reachable from every worker — fine for one team, fatal when the agent itself wants to spin up the topology on demand. **CloudAMQP** and **Upstash Kafka** solve provisioning but cost real money per durable consumer and gate creation behind an account. The **persistence problem**: pairing the queue with a results store usually means a second signup (Supabase, Render, RDS) and a second secret to plumb into every worker. instanode hands the planner, all 12 workers, and the reducer one claimed token that owns a real NATS JetStream URL with subject filtering and a Postgres for findings — both reachable over the public internet, both provisioned in two POSTs. The per-subject filter (`research.done.<run_id>`) means you can run ten of these concurrently in the same stream without correlation-id juggling, and durable consumers mean a worker crash redelivers rather than drops.

## Related cases

- [CrewAI parallel-process crew](/use-cases/crewai-parallel-process-crew) — framework-equivalent parallel-research pattern
- [Magentic-One DAG executor](/use-cases/magentic-one-dag-executor) — explicit DAG version of fan-out research
- [Scatter-gather price comparison swarm](/use-cases/scatter-gather-price-comparison-swarm) — first-N-wins variant of the same fan-out idea
