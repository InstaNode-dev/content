---
title: Adversarial red-team runner
category: P. Agent benchmarking & evaluation
services: ["mongo", "webhook"]
scenario: A LangWatch-Scenario-style service spawns attacker agents that probe a target chatbot in parallel; transcripts persist in Mongo and successful jailbreaks fire a webhook to the security team.
---

## Sample agent prompt

```
You run a LangWatch-Scenario-style red-team. Spawn 50 attacker agents that probe a target chatbot with jailbreak templates. Persist every transcript to MongoDB. When an attacker succeeds (target outputs forbidden content), fire a webhook to the security team's incident channel with the transcript URL.
```

## Steps to follow

- **Step 1: Provision Mongo + a webhook receiver.**

  ```bash
  MONGO_URL=$(curl -sX POST https://api.instanode.dev/nosql/new -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .connection_url)
  WEBHOOK=$(curl -sX POST https://api.instanode.dev/webhook/new -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .receive_url)
  ```

- **Step 2: Define attacker run document.**

  ```python
  from pymongo import MongoClient
  db = MongoClient(MONGO_URL).redteam
  db.runs.create_index([("attacker_id", 1), ("started_at", -1)])
  db.runs.create_index("verdict")
  ```

- **Step 3: Fan out 50 attackers.** Each writes its transcript.

  ```python
  async def attack(template_id):
      transcript = []
      for turn in run_attack(target_url, template_id):
          transcript.append(turn)
      verdict = judge(transcript)  # "blocked" | "leaked" | "partial"
      run_id = db.runs.insert_one({
          "attacker_id": template_id,
          "transcript": transcript,
          "verdict": verdict,
          "started_at": datetime.utcnow(),
      }).inserted_id
      if verdict == "leaked":
          requests.post(WEBHOOK, json={"run_id": str(run_id), "template": template_id})
  ```

- **Step 4: Security team polls the webhook for hits.**

  ```bash
  curl -s "https://api.instanode.dev/api/v1/webhooks/$TOKEN/requests?since=1h" | jq '.[] | .body'
  ```

## Why this works on instanode.dev

Red-team transcripts are bursty, unstructured, and need cheap append-only writes — exactly what Mongo's good at. Pairing it with a webhook means your incident pipeline doesn't need a polling worker; successful jailbreaks push themselves to wherever the on-call lives. Both resources are real (not mocks), so the same setup works for nightly CI runs and one-off ad-hoc probes.
