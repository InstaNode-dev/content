---
title: EHR appointment webhook fan-in
category: C. Vertical AI apps
services: ["webhook", "nats"]
scenario: A medical-intake agent receives appointment-created webhooks from multiple EHR vendors and unifies them into one queue.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a medical-intake agent receives appointment-created webhooks from multiple EHR vendors and unifies them into one queue.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build an intake aggregator. Claim a webhook receiver and a NATS stream on instanode.dev. Register the webhook URL with each EHR vendor's appointment-created hook (Epic, Cerner, Athena). The aggregator normalizes payloads into a canonical {patient_id, ts, location} schema and publishes to NATS subject appointments.created. Downstream agents subscribe.
```

## Steps to follow

- **Step 1: Provision webhook + queue.** Two curls.

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new | jq -r .connection_url)
  echo "Register this URL with each EHR vendor: $WH"
  ```

- **Step 2: Aggregator polls webhook + normalizes.** Vendor schemas differ; canonical does not.

  ```python
  events = httpx.get(f"https://api.instanode.dev/api/v1/webhooks/{token}/requests").json()
  for e in events:
      canonical = normalize(e["headers"]["x-vendor"], e["body"])
      js.publish("appointments.created", json.dumps(canonical).encode())
  ```

- **Step 3: Define the normalizer.** Switch on vendor header.

  ```python
  def normalize(vendor, body):
      if vendor == "epic":
          return {"patient_id": body["MRN"], "ts": body["ApptDateTime"], "location": body["DeptId"]}
      if vendor == "cerner":
          return {"patient_id": body["patient"]["id"], "ts": body["startDateTime"], "location": body["facility"]}
  ```

- **Step 4: Downstream intake agent subscribes.** Independent of vendor.

  ```python
  await js.subscribe("appointments.created", cb=handle_intake)
  ```

## Why this works on instanode.dev

The webhook URL is HIPAA-grade only after a BAA, but for staging or PHI-scrubbed pilots, the public HTTPS endpoint means EHR vendors can register your URL without you owning a load balancer. NATS gives the downstream a buffered, replayable feed instead of synchronous failure under EHR push spikes. Both resources are token-isolated; no shared queue with another tenant.
