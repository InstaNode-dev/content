---
title: Devin-style PR-bot fleet
category: L. Agent-factory / spawning patterns
services: ["pg", "webhook", "deploy"]
scenario: A Cognition-Devin-style orchestrator spawns one fresh agent worker per inbound GitHub issue, each with its own scratch Postgres for running migrations against, and reports back via webhook.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Cognition-Devin-style orchestrator spawns one fresh agent worker per inbound GitHub issue, each with its own scratch Postgres for running migrations against, and reports back via webhook.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + webhook receiver + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
I'm building a Cognition-Devin-style PR bot for a 220k-LOC Rails monorepo at github.com/acme/billing-core. When a maintainer labels an issue `agent:try`, the orchestrator should spawn one isolated worker that owns a fresh Postgres (so it can pg_restore a 1.8GB anonymized prod snapshot and dry-run migrations without touching staging), a webhook receiver (so it can stream progress back to the orchestrator without exposing its own port), and a deploy URL (so the resulting branch ships a preview environment the reviewer can poke). Cap the fleet at 20 concurrent workers; on PR merge or close, reap all three resources for that worker. Show me the orchestrator loop and one worker pass on issue #4127 ("backfill stripe_customer_id on orders").
```

## Steps to follow

We'll thread one issue — **acme/billing-core#4127, "backfill stripe_customer_id on orders"** — through every step. By the end you'll have a spawned worker, a tested migration, an open PR with a preview URL, and a clean reap.

- **Step 1: GitHub webhook → orchestrator → spawn three resources.** Each provision requires a `name`. The orchestrator stores each resource's `id` (returned in the provision response) in its own DB so it can reap later via `DELETE /api/v1/resources/:id`.

  ```bash
  # spawn-worker.sh — called by the orchestrator on `issue.labeled` with agent:try
  ISSUE=$1   # e.g. 4127
  REPO=$2    # e.g. acme/billing-core
  H='Content-Type: application/json'

  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "$H" -d "{\"name\":\"pr-${ISSUE}-scratch-db\"}")
  PG_URL=$(echo $PG | jq -r .connection_url); PG_ID=$(echo $PG | jq -r .id)

  WH=$(curl -sX POST https://api.instanode.dev/webhook/new -H "$H" -d "{\"name\":\"pr-${ISSUE}-callback\"}")
  WH_URL=$(echo $WH | jq -r .receive_url); WH_ID=$(echo $WH | jq -r .id)

  # /deploy/new is multipart — pass name + image + env.* as form fields, JWT as Bearer.
  DEPLOY=$(curl -sX POST https://api.instanode.dev/deploy/new \
    -H "Authorization: Bearer $INSTANODE_TOKEN" \
    -F "name=pr-${ISSUE}-worker" \
    -F "image=ghcr.io/acme/devin-worker:latest" \
    -F "subdomain=pr-${ISSUE}" \
    -F "env.PG_URL=$PG_URL" \
    -F "env.CALLBACK=$WH_URL" \
    -F "env.ISSUE=$ISSUE" \
    -F "env.REPO=$REPO" \
    -F "env.GH_TOKEN=$ORCH_GH_TOKEN")
  DEPLOY_URL=$(echo $DEPLOY | jq -r .url); DEPLOY_ID=$(echo $DEPLOY | jq -r .id)

  psql "$ORCH_DB" -c "INSERT INTO workers (issue, pg_id, wh_id, deploy_id, deploy_url) \
                       VALUES ($ISSUE, '$PG_ID', '$WH_ID', '$DEPLOY_ID', '$DEPLOY_URL');"
  echo "worker spawned for #$ISSUE → $DEPLOY_URL"
  ```

- **Step 2: The worker reads the issue, writes a migration, tests it on its scratch DB.** Throwaway means it can `pg_restore` real prod-shaped data and ROLLBACK without contaminating anything.

  ```bash
  # inside the worker container
  pg_restore --no-owner --no-acl -j 4 -d "$PG_URL" /seed/anonymized-prod-2026-05-10.dump

  git clone --depth 50 "https://x-access-token:$GH_TOKEN@github.com/$REPO" /work && cd /work
  git checkout -b agent/$ISSUE

  # the agent writes db/migrate/20260511_backfill_stripe_customer_id.rb,
  # then validates against the scratch DB:
  ```

  ```sql
  -- inside psql against $PG_URL
  BEGIN;
  \i db/migrate/20260511_backfill_stripe_customer_id.sql
  SELECT count(*) FILTER (WHERE stripe_customer_id IS NULL) AS still_null
    FROM orders;
  -- still_null = 0  ✓
  EXPLAIN ANALYZE SELECT * FROM orders WHERE stripe_customer_id = $1;
  ROLLBACK;
  ```

- **Step 3: Worker opens the PR and POSTs progress back via its private webhook.** The orchestrator never has to expose an inbound port — it polls `/api/v1/webhooks/<token>/requests` for delivered messages.

  ```python
  # worker/report.py
  import httpx, os
  pr = open_pr(
      repo=os.environ["REPO"],
      branch=f"agent/{os.environ['ISSUE']}",
      title="Backfill stripe_customer_id on orders (agent:try)",
      body=f"Preview: {os.environ['DEPLOY_URL']}\nDry-run: 0 NULL rows after migration; index plan attached.",
  )
  httpx.post(os.environ["CALLBACK"], json={
      "issue":   int(os.environ["ISSUE"]),
      "pr":      pr["html_url"],
      "preview": os.environ["DEPLOY_URL"],
      "status":  "ready_for_review",
  })
  ```

- **Step 4: Orchestrator polls the webhook receiver, updates GitHub.** instanode's webhook resource stores every received POST; the orchestrator pulls them on a 5s tick.

  ```bash
  # orchestrator/poll-loop.sh
  while true; do
    psql "$ORCH_DB" -c "SELECT issue, wh_token FROM workers WHERE status='spawned';" -t -A -F'|' \
    | while IFS='|' read ISSUE WH; do
        curl -s "https://api.instanode.dev/api/v1/webhooks/$WH/requests?since=last" \
          | jq -c '.[] | select(.body.status=="ready_for_review")' \
          | while read msg; do
              PR=$(echo $msg | jq -r .body.pr)
              gh issue comment $ISSUE --repo $REPO --body "Agent PR ready: $PR"
              psql "$ORCH_DB" -c "UPDATE workers SET status='pr_open', pr_url='$PR' WHERE issue=$ISSUE;"
            done
      done
    sleep 5
  done
  ```

- **Step 5: On PR close/merge, reap all three resources for this worker.** Anonymous-tier auto-reap (24h) is a backstop if the orchestrator's reaper is buggy — a property a self-hosted fleet would never get for free. Teardown is `DELETE /api/v1/resources/:id` with the orchestrator's claimed-session Bearer token.

  ```bash
  # reap.sh — called by GitHub webhook on pull_request.closed
  ISSUE=$1
  read PG_ID WH_ID DEPLOY_ID <<< $(psql "$ORCH_DB" -t -A -F' ' \
      -c "SELECT pg_id, wh_id, deploy_id FROM workers WHERE issue=$ISSUE;")

  for RID in "$PG_ID" "$DEPLOY_ID" "$WH_ID"; do
    curl -sX DELETE "https://api.instanode.dev/api/v1/resources/$RID" \
      -H "Authorization: Bearer $INSTANODE_TOKEN"
  done

  psql "$ORCH_DB" -c "UPDATE workers SET status='reaped', reaped_at=now() WHERE issue=$ISSUE;"
  echo "reaped worker for #$ISSUE"
  ```

- **Step 6: Verify end-to-end.** Spawn against #4127, watch the fleet table.

  ```bash
  ./spawn-worker.sh 4127 acme/billing-core
  sleep 60
  psql "$ORCH_DB" -c "SELECT issue, status, pr_url, deploy_url FROM workers WHERE issue=4127;"
  #  issue | status         | pr_url                                | deploy_url
  # -------+----------------+---------------------------------------+--------------------------------------
  #   4127 | pr_open        | https://github.com/acme/.../pull/4131 | https://pr-4127.instanode.app
  ```

## Why this works on instanode.dev

Devin-style fleets all hit the same wall: per-worker isolation is the entire point of the pattern, and every component of that isolation has historically required its own provisioning story. The scratch DB needs to be a real Postgres (not SQLite — the migration uses `pg_depend` and partial indexes), which on **AWS RDS** is 4–7 minutes and an IAM role per worker; on **Neon** it's faster but the API gates project creation per dashboard plan; on **Supabase** it's per-project, not per-resource, so 20 workers means 20 dashboard projects. The preview URL would normally route through **Vercel** or **Fly.io** preview environments — each requiring an org/app setup the orchestrator's service account has to be entitled for. The webhook receiver is usually a self-hosted endpoint (which means **ngrok** tunnels in dev and an actual public service in prod, plus signature plumbing). instanode collapses these into three POSTs per worker. The same JWT owns all three resources, so the reap path is three DELETEs from one credential — and if the reaper is buggy, anonymous-tier auto-expiry in 24h limits the blast radius to a single day's worth of orphaned resources, not a forever-growing dashboard of forgotten Neon projects. The preview is real DNS on `*.instanode.app`, so PR comments can include a clickable link the reviewer pastes directly.

## Related cases

- [High-volume PR-review pipeline](/use-cases/high-volume-pr-review-pipeline.md) — queue-based equivalent at thousands-of-PRs scale
- [Cursor background-agent worktree](/use-cases/cursor-background-agent-worktree.md) — IDE-side counterpart that owns a worktree per branch
- [Sandbox-per-PR preview deployment](/use-cases/sandbox-per-pr-preview-deployment.md) — the preview-env primitive this PR-bot fleet calls into
