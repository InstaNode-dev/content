# instanode.dev — public content

Source of truth for the marketing site (https://instanode.dev) content.

The dashboard repo (`InstaNode-dev/instanode-web`) clones this repo into
its `.content/` directory at build time and inlines the markdown into
the static HTML via Vite's `import.meta.glob` raw imports. The same
flow runs in `npm run dev` so authors get hot reload on edits in a
local clone.

## Layout

```
.
├── blog/
│   ├── <slug>.md          one post per file; filename = URL slug
│   └── ...
├── use-cases.json         (slice B — coming)
└── docs/                  (slice C — coming)
    └── <section>.md
```

## Blog post format

```markdown
---
title: Maya shipped on Sunday night
date: 2026-05-12
author: instanode.dev
excerpt: One-line summary used in the /blog index card and meta tags.
---

# Maya shipped on Sunday night

Body in CommonMark. The dashboard renderer supports a minimal subset —
H1–H3, paragraphs, fenced code blocks, unordered lists, inline code,
bold. No HTML passthrough (XSS safety).
```

Frontmatter rules:
- `title`, `date` (ISO `YYYY-MM-DD`), `author`, `excerpt` — required
- All values are single-line strings — no YAML block scalars
- Slug = filename without `.md`

## Editing flow

1. `git clone git@github.com:InstaNode-dev/content.git`
2. Edit a `.md` file. Single line edits ship the same way structural ones do.
3. Open a PR. Reviewers see only the diff to the prose — no React, no CSS, no build config.
4. Merge. The next dashboard build picks up the change.

## What NOT to put here

This repo is **public**. Everything in it ships to the public marketing
domain. Do not commit:

- Internal cluster hostnames (`*.svc.cluster.local`)
- Secret variable names (`AES_KEY`, `JWT_SECRET`, `RAZORPAY_*`)
- Customer team IDs, real user emails, real production identifiers
- Stack traces from production
- Roadmap items that haven't been publicly announced

If a piece of content is engineering-internal, it belongs in the
private `InstaNode-dev/docs` repo instead.
