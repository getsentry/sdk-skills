---
name: contributing-md
description: >
  Generate or align a CONTRIBUTING.md for a Sentry SDK repository with the
  develop.sentry.dev standard template. Use when asked to "create contributing.md",
  "update contributing.md", "align contributing docs", "standardize contributor docs",
  or when the contributing guide is outdated, missing sections, or does not exist.
---

# Contributing.md Skill

Generate or align a Sentry SDK repository's `CONTRIBUTING.md` with the
[standard template](https://develop.sentry.dev/sdk/getting-started/templates/contributing-md-template/).
Spec compliance is the primary goal; SDK-specific content is preserved or added around it.
Target **100–200 lines** — detailed process lives on develop.sentry.dev, link there, don't duplicate.

## Steps

### 1. Fetch the template

```
https://develop.sentry.dev/sdk/getting-started/templates/contributing-md-template/
```

This is the source of truth for required sections, structure, and links.

### 2. Read the existing file (if any)

If `CONTRIBUTING.md` exists, read it. Commands and SDK-specific content already there
should be trusted. If it does not exist, use the fetched template as the starting point.

### 3. Find commands from the repo

For missing or placeholder values, read in priority order:

1. `.github/workflows/*.yml` — authoritative test/lint/build commands
2. `Makefile` / `Taskfile.yml` / `tox.ini` — command wrappers
3. `README.md` — install command, SDK name
4. `docs.sentry.io` for the SDK — setup steps if not found above

### 4. Reference good examples

Check these repos for structural inspiration if needed:

- [`sentry-python`](https://github.com/getsentry/sentry-python/blob/master/CONTRIBUTING.md) — integration principles, user vs contributor setup
- [`sentry-javascript`](https://github.com/getsentry/sentry-javascript/blob/master/CONTRIBUTING.md) — Volta pin, monorepo workflow, draft PR rule
- [`sentry-go`](https://github.com/getsentry/sentry-go/blob/master/CONTRIBUTING.md) — LOGAF usage, craft release-notes requirement
- [`sentry-cocoa`](https://github.com/getsentry/sentry-cocoa/blob/master/CONTRIBUTING.md) — LOGAF with descriptions, copyright header rule
- [`sentry-java`](https://github.com/getsentry/sentry-java/blob/master/CONTRIBUTING.md) — API compatibility workflow, step-by-step setup

### 5. Write the file

- Follow the section order from the fetched template
- Append preserved SDK-specific sections after standard content
- Flag contradictions rather than removing:
  ```html
  <!-- TODO: review against standard: https://develop.sentry.dev/... -->
  ```
- If file would exceed 200 lines, move verbose steps to a linked doc and reference it

### 6. Verify

Grep for `\[` — no unreplaced placeholder tokens should remain.

## Quality Checklist

- [ ] All `[placeholder]` values replaced with real content
- [ ] Commands verified against CI config or README
- [ ] Existing SDK-specific sections preserved
- [ ] Non-standard content flagged with TODO comments
- [ ] AI attribution section present
- [ ] File is 100–200 lines
- [ ] No links point to Notion or Linear
