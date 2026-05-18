---
title: Pre-commit skill-scanner webhook
category: F. Developer tooling
services: ["webhook"]
scenario: An MCP skill security scanner receives pre-commit webhooks, scans agent skills against a ruleset, and blocks the push on critical findings.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an MCP skill security scanner receives pre-commit webhooks, scans agent skills against a ruleset, and blocks the push on critical findings.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Set up a pre-commit MCP-skill security scanner. Provision a webhook endpoint; configure husky/pre-commit hooks in our repos to POST the staged skill files. The scanner runs a rule set (shell out, network egress, unbounded eval) and replies non-200 to block on critical findings. Return the URL.
```

## Steps to follow

- **Step 1: Provision the webhook.**

  ```bash
  curl -s -X POST https://api.instanode.dev/webhook/new -H 'Content-Type: application/json' -d '{"name":"pre-commit-skill-scanner-webhook-webhook"}' | jq -r .receive_url
  ```

- **Step 2: Wire the pre-commit hook.**

  ```bash
  cat > .git/hooks/pre-commit <<'SH'
  #!/usr/bin/env bash
  staged=$(git diff --cached --name-only --diff-filter=AM | grep -E '\.claude/skills/.*\.md$')
  [ -z "$staged" ] && exit 0
  payload=$(jq -n --arg files "$(tar -czf - $staged | base64)" '{archive: $files}')
  code=$(curl -s -o /tmp/scan.json -w "%{http_code}" -XPOST "$WH_URL" -d "$payload")
  [ "$code" = "200" ] || { jq . /tmp/scan.json; exit 1; }
  SH
  chmod +x .git/hooks/pre-commit
  ```

- **Step 3: Scanner pulls submissions and applies the rule set.**

  ```python
  for sub in fetch_webhook_batch(WH_URL):
      findings = []
      for path, body in untar(sub["archive"]):
          findings += run_rules(body, rules=["shell_out", "egress", "eval"])
      verdict = "block" if any(f.severity == "critical" for f in findings) else "ok"
  ```

- **Step 4: Reply with verdict on the request the hook polls.**

  ```python
  # Hook polls /requests/{id} until verdict appears, then exit 0/1 accordingly.
  ```

- **Step 5: CI fallback** runs the same scan on PR-open so local-hook bypass still gets caught.

  ```yaml
  - run: ./.github/scripts/run-skill-scan.sh
  ```

## Why this works on instanode.dev

The webhook receiver gives the scanner a single ingress URL that every dev's hook can POST to, with no per-developer auth setup. Rules run server-side so your scanner improvements ship instantly without each developer running `pip install --upgrade`.

## Related cases

- [Adversarial red-team runner](/use-cases/adversarial-red-team-runner) — runtime attack-surface counterpart to this static scanner
- [PR-review bot triggered by webhooks](/use-cases/pr-review-bot-triggered-by-webhooks) — another webhook-triggered code-quality bot
- [SARIF scan-result store](/use-cases/sarif-scan-result-store) — Postgres warehouse where scanner findings can land for trends
