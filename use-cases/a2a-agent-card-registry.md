---
title: A2A agent-card registry
category: G. Internet-of-AI
services: ["pg"]
scenario: A marketplace stores Agent Cards (skills, tags, pricing) and serves discovery queries to client agents.
---

## Sample agent prompt

```
You're building an A2A discovery service. Provision a Postgres on instanode.dev, store Agent Cards (id, name, skills array, tags, pricing JSON), and expose a `/discover` query that filters by skill tag and returns ranked cards. Index skills for sub-50ms lookups across 100k cards.
```

## Steps to follow

- **Step 1: Provision Postgres.** Use a claimed token so cards persist beyond 24h.

  ```bash
  curl -sX POST https://api.instanode.dev/db/new \
    -H "Authorization: Bearer $INSTANT_TOKEN" | jq .
  ```

- **Step 2: Define the agent_cards table with GIN-indexed skills.**

  ```sql
  CREATE TABLE agent_cards (
    id text PRIMARY KEY,
    name text NOT NULL,
    skills text[] NOT NULL,
    tags text[],
    pricing jsonb,
    endpoint text NOT NULL,
    registered_at timestamptz DEFAULT now()
  );
  CREATE INDEX idx_skills ON agent_cards USING GIN (skills);
  CREATE INDEX idx_tags ON agent_cards USING GIN (tags);
  ```

- **Step 3: Register an Agent Card.** A2A spec calls this the `/.well-known/agent.json` payload.

  ```sql
  INSERT INTO agent_cards (id, name, skills, tags, pricing, endpoint)
  VALUES ('travel-pro-v2', 'TravelPro',
          ARRAY['flights','hotels','itinerary'],
          ARRAY['leisure','business'],
          '{"per_call_usd": 0.02}'::jsonb,
          'https://travelpro.example.com/a2a');
  ```

- **Step 4: Discovery query — find agents that do "flights" under 5 cents per call.**

  ```sql
  SELECT id, name, endpoint, pricing
  FROM agent_cards
  WHERE 'flights' = ANY(skills)
    AND (pricing->>'per_call_usd')::numeric < 0.05
  ORDER BY registered_at DESC LIMIT 20;
  ```

## Why this works on instanode.dev

A2A registries need real relational filtering across array columns — KV stores can't express "skills contains X AND pricing.per_call_usd < Y". You get a real Postgres in one curl, GIN-indexed for the only access pattern that matters, with no per-row registry fees and no vendor lock-in to a proprietary directory service.
