---
title: Authentication
order: 6
---

Resource provisioning is anonymous. Everything else (deploy, vault, billing,
team management) requires a session JWT.

How to get one:

1. Provision any resource anonymously. The response includes a JWT in the
   `upgrade_jwt` field.
2. POST that JWT to /claim with an email. We send a magic link.
3. Click the link in the email; the page sets a session cookie.

For unattended use (CI, agents), exchange the session cookie for a long-lived
API key at `POST /api/v1/api-keys`. Pass it as `Authorization: Bearer
<key>` on every request.

To verify a token works at any time, hit `GET /api/v1/whoami` — returns
200 with your team_id + plan_tier on success, 401 on failure.
