---
title: Long-horizon Temporal agent workflow
category: Q. Background/async agent fleets
services: ["pg", "nats"]
scenario: A Temporal-backed agent runs a 14-day procurement negotiation across vendors; each step persists in Postgres with NATS signals so a human can interject without breaking the workflow.
---

## Sample agent prompt

```
Set up a Temporal-backed 14-day procurement negotiation agent. Each negotiation step persists to Postgres and emits a NATS signal so a human can intervene. Provision Postgres for negotiation state and NATS for signal delivery, then return URLs we wire into the Temporal worker.
```

## Steps to follow

- **Step 1: Provision the persistent state store and signal bus.**

  ```bash
  curl -s -X POST https://api.instanode.dev/db/new
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"stream":"procurement"}' -H 'Content-Type: application/json'
  ```

- **Step 2: Negotiation state schema.**

  ```sql
  CREATE TABLE negotiation_steps (
    workflow_id text NOT NULL,
    step int NOT NULL,
    vendor text,
    offer numeric,
    counter numeric,
    decided_at timestamptz,
    PRIMARY KEY (workflow_id, step)
  );
  ```

- **Step 3: Temporal activity persists each step.**

  ```python
  @activity.defn
  async def record_offer(wf_id, step, vendor, offer):
      pg.execute("INSERT INTO negotiation_steps(workflow_id,step,vendor,offer) VALUES (%s,%s,%s,%s)",
                 wf_id, step, vendor, offer)
      await nc.publish("procurement.offer", json.dumps({"wf": wf_id, "offer": offer}).encode())
  ```

- **Step 4: Human-interject signal handler in the workflow.**

  ```python
  @workflow.signal
  async def human_override(self, new_terms):
      self.terms = new_terms
  ```

- **Step 5: Operator UI publishes an override via NATS, which the worker translates to a Temporal signal.**

  ```bash
  nats pub procurement.override '{"wf":"proc-42","floor":18000}'
  ```

## Why this works on instanode.dev

Temporal handles the 14-day timer; Postgres holds the negotiation audit trail; NATS carries human interjections. All three are provisioned in two curls, and the same Postgres can host Temporal's persistence backend if you want to consolidate further.
