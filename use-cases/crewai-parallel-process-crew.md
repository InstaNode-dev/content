---
title: CrewAI parallel-process crew
category: J. Agent swarms & fan-out
services: ["pg", "redis"]
scenario: A CrewAI Process.parallel run executes researcher, writer, and fact-checker agents simultaneously against a shared pgvector store, each persisting partial drafts as it works.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a CrewAI Process.parallel run executes researcher, writer, and fact-checker agents simultaneously against a shared pgvector store, each persisting partial drafts as it works.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Configure CrewAI Process.parallel for a researcher + writer + fact-checker. Use instanode.dev Postgres with pgvector as the shared embedding store and Redis as the partial-draft cache. Each agent appends its partial to Redis so the others see progress as it streams.
```

## Steps to follow

- **Step 1: Provision Postgres + Redis.** Same anonymous token claims both; no separate signups.

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"crewai-parallel-process-crew-db"}' | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"crewai-parallel-process-crew-cache"}' | jq -r .connection_url)
  ```

- **Step 2: Set up the shared pgvector store.** All three agents read from this single table.

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE corpus (id bigserial PRIMARY KEY, source text, chunk text, embedding vector(1536));
  ```

- **Step 3: Run the crew in parallel.** Each agent's task pushes its partial draft to a Redis key.

  ```python
  from crewai import Crew, Process
  crew = Crew(agents=[researcher, writer, fact_checker], process=Process.parallel)

  def on_partial(agent_name, chunk):
      r.append(f"draft:{run_id}:{agent_name}", chunk)
  crew.kickoff(inputs={"topic": "RAG eval"}, on_chunk=on_partial)
  ```

- **Step 4: Stitch partials.** Read all three keys at the end; reconcile via embedding similarity.

  ```python
  parts = {a: r.get(f"draft:{run_id}:{a}") for a in ["researcher","writer","fact_checker"]}
  ```

## Why this works on instanode.dev

Anonymous claim gives you both services in the same flow, so the CrewAI `tools/` list can hardcode connection URLs once. Redis is small but fast — perfect for streaming partials where state is overwritten constantly — while Postgres holds the durable embedding store. No queue infrastructure required; partial visibility is just a Redis SUBSCRIBE.

## Related cases

- [LangGraph fan-out research agents](/use-cases/langgraph-fan-out-research-agents) — framework-equivalent parallel-research pattern with NATS
- [OpenAI Agents SDK handoff mesh](/use-cases/openai-agents-sdk-handoff-mesh) — specialist-agent fan-out in the OpenAI SDK shape
- [Magentic-One DAG executor](/use-cases/magentic-one-dag-executor) — explicit DAG variant of the same parallel-crew idea
