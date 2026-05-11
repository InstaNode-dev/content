---
title: Classroom-per-student sandbox
category: I. Hackathon & education
services: ["pg"]
scenario: A CS professor provisions one ephemeral Postgres per student for a SQL assignment, dropped after grading.
---

## Sample agent prompt

```
You're a CS professor running a SQL assignment for 40 students. Provision one ephemeral Postgres per student, seed it with the assignment fixture data, hand out connection URLs. After grading on Friday, let them expire — no manual teardown.
```

## Steps to follow

- **Step 1: Loop the roster, provision per student.**

  ```bash
  while read student email; do
    url=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
    echo "$student,$email,$url" >> handouts.csv
  done < roster.csv
  ```

- **Step 2: Seed each DB with the assignment fixture.**

  ```bash
  while IFS=, read student email url; do
    psql "$url" -f assignment_seed.sql &
  done < handouts.csv
  wait
  ```

- **Step 3: Email each student their personal URL.** The assignment uses standard SQL — they connect from psql, DataGrip, anything.

  ```python
  for row in csv.DictReader(open("handouts.csv")):
      send_mail(row["email"], subject="CS 348 Assignment 3 — your DB",
                body=f"psql {row['url']}\n\nWork through problems 1-10.")
  ```

- **Step 4: After grading, do nothing.** Anonymous resources auto-expire at T+24h; for a 7-day variant use claimed tokens with a manual delete after grading.

  ```bash
  # Optional explicit teardown:
  curl -X DELETE "https://api.instanode.dev/api/v1/resources/$id" \
    -H "Authorization: Bearer $TEACHER_TOKEN"
  ```

## Why this works on instanode.dev

40 fresh Postgres instances for one assignment is comically expensive on RDS and operationally annoying on shared schemas (students nuke each other's tables). Per-student isolated databases via one curl each — total provisioning cost: zero, total teardown cost: zero. Students can also `DROP DATABASE` in their own without affecting peers.
