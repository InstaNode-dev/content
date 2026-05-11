---
title: SARIF scan-result store
category: F. Developer tooling
services: ["pg"]
scenario: A security scanner posts SARIF results into Postgres so trend dashboards show drift over weeks of commits.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a security scanner posts SARIF results into Postgres so trend dashboards show drift over weeks of commits.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Every nightly run of Semgrep and CodeQL on the main branch should POST its SARIF output to a Postgres table provisioned via instanode.dev. Build a small ingester that flattens SARIF results into rows keyed by (rule_id, commit_sha, file, line), so the dashboard can plot drift over weeks.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Create the flat findings table.**

  ```sql
  CREATE TABLE findings (
    id BIGSERIAL PRIMARY KEY,
    scanner TEXT, commit_sha TEXT, rule_id TEXT,
    severity TEXT, file TEXT, line INT,
    message TEXT, scanned_at TIMESTAMPTZ DEFAULT now()
  );
  CREATE INDEX ON findings (commit_sha);
  CREATE INDEX ON findings (rule_id, scanned_at);
  ```

- **Step 3: Flatten and bulk-insert SARIF.**

  ```python
  import json, psycopg
  sarif = json.load(open("semgrep.sarif"))
  rows = [(r["ruleId"], commit, loc["physicalLocation"]["artifactLocation"]["uri"],
           loc["physicalLocation"]["region"]["startLine"], r["message"]["text"])
          for run in sarif["runs"] for r in run["results"]
          for loc in r["locations"]]
  with psycopg.connect(DATABASE_URL) as c:
      c.cursor().executemany(
        "INSERT INTO findings(rule_id,commit_sha,file,line,message) VALUES (%s,%s,%s,%s,%s)", rows)
  ```

- **Step 4: Trend query.**

  ```sql
  SELECT date_trunc('day', scanned_at) d, rule_id, count(*)
  FROM findings GROUP BY 1,2 ORDER BY 1 DESC;
  ```

- **Step 5: Wire the dashboard** to that query and the agent can answer "are we getting better or worse on XSS findings".

## Why this works on instanode.dev

A real Postgres on one curl removes the awkward step where security tools store results in a CI artifact and you can never query history. JSONB columns also handle the raw SARIF payload for downstream tooling that needs the original.
