---
title: Token-cost ledger per sub-agent
category: N. Multi-agent observability
services: ["webhook", "pg", "redis"]
scenario: Every LLM call from any agent in a fleet emits a webhook with input/output tokens; a ledger agent aggregates spend per tenant in Postgres and triggers throttling at thresholds.
---

## Sample agent prompt

```
Every LLM call across the agent fleet POSTs token usage to a webhook provisioned via instanode.dev. A ledger consumer drains the webhook, increments per-tenant counters in Redis, and persists daily totals to Postgres. If a tenant exceeds $50/day in Redis, set tenant:<id>:throttled and have agents check it before each call.
```

## Steps to follow

- **Step 1: Provision webhook + Redis + Postgres.**

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  DB=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  ```

- **Step 2: Agents emit usage.**

  ```python
  requests.post(WH, json={
    "tenant": tenant_id, "agent": agent_id,
    "input": resp.usage.input_tokens, "output": resp.usage.output_tokens,
    "model": "claude-opus-4-7"
  })
  ```

- **Step 3: Drain + increment Redis counters.**

  ```python
  for evt in drain_webhook():
      cents = price(evt["model"], evt["input"], evt["output"])
      total = r.incrby(f"spend:{evt['tenant']}:{today}", cents)
      if total > 5000: r.setex(f"throttled:{evt['tenant']}", 3600, "1")
  ```

- **Step 4: Daily roll-up to Postgres.**

  ```sql
  INSERT INTO daily_spend(tenant_id, day, cents)
  VALUES ($1, $2, $3) ON CONFLICT (tenant_id, day) DO UPDATE SET cents = EXCLUDED.cents;
  ```

- **Step 5: Agent precheck.**

  ```python
  if r.get(f"throttled:{tenant_id}"): raise BudgetExceeded()
  ```

## Why this works on instanode.dev

The webhook captures every call even if the ledger consumer is briefly down — replay drains on restart. Redis `INCRBY` is atomic so concurrent sub-agents can't race past the threshold. Postgres holds the audit trail for invoicing.
