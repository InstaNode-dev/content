---
title: Deploying an app
order: 3
---

`POST /deploy/new` takes a multipart form with a gzipped tar archive
containing your Dockerfile + source.

```
curl -X POST https://api.instanode.dev/deploy/new \
  -H "Authorization: Bearer <JWT>" \
  -F "tarball=@app.tar.gz" \
  -F "name=expense-tracker" \
  -F "port=8080" \
  -F 'env_vars={"DATABASE_URL":"postgres://..."}'
```

`name` is **required** — it is the human-readable label for the deployment.
Send it as a form field. It must be 1–64 characters and match
`^[A-Za-z0-9][A-Za-z0-9 _-]*$` (start with a letter or digit; letters,
digits, spaces, underscores and hyphens after). Omitting it returns
`400 {"error":"name_required"}`; an invalid value returns
`400 {"error":"invalid_name"}`.

The build runs in-cluster on kaniko (~30–90s for typical Node/Python apps)
and the app rolls out behind a public HTTPS URL on
`*.deployment.instanode.dev` with a valid Let's Encrypt cert.

`env_vars` is optional — pass a JSON object and every key/value lands in
the app's environment on the first build. Saves you a follow-up PATCH+redeploy.

For multi-service apps see **Stacks** below.

## Deleting a deployment (paid tiers — two-step, email-confirmed)

Paid customers (Hobby, Hobby Plus, Pro, Growth, Team) can free a consumed
deployment slot at any time. Because deletion is destructive — every byte
of the running app + every env var — the agent CAN initiate but CANNOT
finalise destruction. Only the human, by clicking the email link, completes
the deletion.

### Step 1 — Initiate

```
curl -X DELETE https://api.instanode.dev/api/v1/deployments/<id> \
  -H "Authorization: Bearer $INSTANODE_TOKEN"
```

Response (HTTP 202):

```
{
  "ok": true,
  "id": "<deploy_id>",
  "deletion_status": "pending_confirmation",
  "confirmation_sent_to": "m***@instanode.dev",
  "confirmation_expires_at": "2026-05-14T10:30:00Z",
  "agent_action": "Tell the user to check their email at m***@instanode.dev. The deletion link expires in 15 minutes. To free the slot the user must click the link. The agent CANNOT confirm on the user's behalf — only the human can.",
  "cancellation_note": "Cancel by calling DELETE on the /confirm-deletion path, or let the 15-minute window expire."
}
```

The agent surfaces `agent_action` to the user verbatim. The slot stays
**consumed** until confirmation — a fresh `POST /deploy/new` still hits
the per-tier `deployments_apps` ceiling.

### Step 2 — User clicks the email link

The email link points at the API's `/auth/email/confirm-deletion?t=<token>`,
which 302s the user to the dashboard's `/app/confirm-deletion` page. The
dashboard runs the authenticated POST:

```
curl -X POST 'https://api.instanode.dev/api/v1/deployments/<id>/confirm-deletion?token=<plaintext>' \
  -H "Authorization: Bearer $INSTANODE_TOKEN"
```

Response (HTTP 200) → `deletion_status: "confirmed"`, slot is free.

### Cancel, expire, agent-override

- **Cancel** (user changes their mind): `DELETE /api/v1/deployments/<id>/confirm-deletion`. Resource stays active.
- **Expire** (15 minutes elapsed): the worker flips the row to `expired`. Re-running `DELETE` mints a fresh email.
- **Agent override**: set `X-Skip-Email-Confirmation: yes` on the original `DELETE` → 200 immediate destruction. Use only when the agent has obtained explicit user consent on its own side.

### Anonymous tier

Anonymous resources (24h TTL) have no email on file. `DELETE` returns
200 immediately — no two-step gate, since there is no inbox to mail.

### Stacks

Same contract applies to `DELETE /api/v1/stacks/<slug>`.
