---
title: Debugging a failed deploy (for AI agents)
order: 5
---

A deploy can fail at build (bad Dockerfile, missing file in the tarball,
dependency error) or roll out but crash at runtime. You do **not** need
cluster access to diagnose it — the platform classifies the failure and
serves the real error back to you over HTTP. This page is written for an
AI agent running a deploy → fix → redeploy loop.

## The auto-debug loop (authenticated deploys via `POST /deploy/new`)

When a deploy you started with `POST /deploy/new` ends up `failed`, run
this loop. The reliable machine surface is `GET /api/v1/deployments/:id/events` —
it is self-contained (reason + last_lines + hint) and needs only your
session token.

1. **Watch the build live (optional).** Stream the build log over SSE while
   it builds:

   ```
   curl -N https://api.instanode.dev/deploy/<id>/logs \
     -H "Authorization: Bearer $INSTANODE_TOKEN"
   ```

2. **Get the one-line status + summary.**

   ```
   curl https://api.instanode.dev/api/v1/deployments/<id> \
     -H "Authorization: Bearer $INSTANODE_TOKEN"
   ```

   Returns `status` (`building` / `failed` / `running` / `expired`) and,
   on failure, `error_message` — a `<reason>: <hint snippet>` summary
   (Kaniko error / ImagePullBackOff / BackoffLimitExceeded / DeadlineExceeded).

3. **Read the classified cause — the real error.** This is the surface to
   act on:

   ```
   curl https://api.instanode.dev/api/v1/deployments/<id>/events \
     -H "Authorization: Bearer $INSTANODE_TOKEN"
   ```

   Returns `{events, count}` where each event is:

   ```json
   {
     "kind": "failure_autopsy",
     "reason": "BackoffLimitExceeded",
     "exit_code": 1,
     "event": "...",
     "last_lines": ["...", "the tail of the build-pod log — the real error output"],
     "hint": "plain-language remedy",
     "created_at": "2026-06-06T..."
   }
   ```

   - `reason` — the classified failure class.
   - `last_lines` — the **tail of the build-pod logs**, the actual compiler /
     installer / Kaniko output that explains the failure. Read this first.
   - `hint` — a plain-language remedy for that reason.

4. **Fix it.** Edit the Dockerfile, the tarball contents, the `port`, or
   the `env_vars` per `hint` + `last_lines`. Common cases: a missing file
   that needed to be in the tar, a build step that needs a dependency, a
   wrong base image, an app that listens on a port other than the one you
   passed.

5. **Redeploy in place** (same `app_id`, same URL, slot count unchanged):

   ```
   curl -X POST https://api.instanode.dev/deploy/<id>/redeploy \
     -H "Authorization: Bearer $INSTANODE_TOKEN"
   ```

   Or pass `redeploy=true` on `POST /deploy/new` with the **same** `name`
   you used originally — the platform rebuilds the existing deployment in
   place and the response carries `"redeployed": true`. (Without
   `redeploy=true` a fresh `POST /deploy/new` mints a NEW app and a NEW
   URL, even when `name` collides.)

6. **Re-verify.** Poll `GET /api/v1/deployments/<id>` until `status` is
   `running` — or loop back to step 3 if it failed again.

## Anonymous deploys (via `POST /stacks/new`)

Anonymous (no-Bearer) callers cannot use `/deploy/new` — they deploy via
`POST /stacks/new` (anonymous stacks carry no team and expire after a 6h
TTL). The failure-diagnosis path for an anonymous stack is **thinner**:

1. **Status + raw error.** Read the stack by its slug (no auth needed —
   the slug is the bearer):

   ```
   curl https://api.instanode.dev/api/v1/stacks/<slug>
   ```

   On failure this returns `status="failed"` plus the raw error string.

2. **Per-service build logs.**

   ```
   curl https://api.instanode.dev/stacks/<slug>/logs/<service>
   ```

Anonymous stacks do **not** have the classified `/events` autopsy
(`reason` / `last_lines` / `hint`) — there is no `/stacks/:slug/events`
endpoint. That is a known thinner path: anonymous deploys get **status +
raw error + logs**, not the classified autopsy. Claim/upgrade to deploy
via `/deploy/new` for the full debug surface.

## Caveats (read these — they affect how you diagnose)

- **Don't rely on the failure email.** A `failed` deploy records a failure
  notification, but transactional email delivery is currently blocked (the
  sender domain isn't validated in prod), so the email may not reach a real
  inbox. Use `GET /api/v1/deployments/:id/events` and the dashboard
  failure-autopsy panel as the source of truth, not email.
- **"Diagnostics pending" window.** For a few seconds right after a
  failure the autopsy is still capturing the build-pod logs — `/events`
  may be empty or carry `reason="Unknown"`. Wait a moment and re-poll.
- **Runtime crash-loops are thinner than build failures.** A deploy that
  *builds* fine but crash-loops at runtime (CrashLoopBackOff, OOMKilled,
  readiness-probe failure) has less customer-facing diagnostics today than
  the build-failure autopsy. Build-failure diagnosis is the
  well-instrumented path; deeper runtime crash-loop visibility is a known
  follow-up.

## Surfaces at a glance

| Surface | What it gives you |
| --- | --- |
| `GET /api/v1/deployments/:id` | `status` + one-line `error_message` |
| `GET /api/v1/deployments/:id/events` | classified `reason` + `last_lines` + `hint` (the real error — use this) |
| `GET /deploy/:id/logs` | live build log stream (SSE) |
| `GET /api/v1/stacks/:slug` | anonymous-stack `status` + raw error string |
| `GET /stacks/:slug/logs/:svc` | anonymous-stack per-service build logs |
