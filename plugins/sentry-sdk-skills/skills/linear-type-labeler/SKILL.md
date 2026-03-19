---
name: linear-type-labeler
description: >
  Classify and apply Type labels to Linear issues based on their title and description.
allowed-tools: mcp__linear-server__query_data mcp__linear-server__get_issue mcp__linear-server__save_issue AskUserQuestion
compatibility: Requires the Linear MCP server to be configured.
---

# Linear Type Labeler

Classifies Linear issues and applies a **Type** label from the Sentry workspace's label
taxonomy based on the content of each issue's title and description.

## Requirements

This skill requires the [Linear MCP server](https://github.com/linear/linear-mcp) to be configured.

## Type Label Reference

Type labels in the Sentry workspace (workspace-level, not per-team):

| Label | When to use |
|---|---|
| **Bug** | Something is broken, regression, error, crash, incorrect behavior |
| **Feature** | New capability, integration, API, SDK support — something that doesn't exist yet |
| **Improvement** | Making existing functionality better — performance, DX, UX, reliability |
| **Task** | Chore, investigation, spike, migration, cleanup, operational work |
| **Tracking Issue** | Parent/umbrella tracking a larger body of work; has sub-issues or a checklist |
| **Docs** | Documentation, RFC, write-up, changelog, guide |
| **Tests** | Writing or fixing tests, coverage, test infrastructure |

## Workflow

### Step 1 — Identify the team

Infer in order: (1) current GitHub repository name — for the `getsentry` org, the repo name
matches the Linear team name, (2) explicitly stated by the user, (3) if still ambiguous, ask.

### Step 2 — Fetch label IDs

Use `query_data` to fetch workspace-level labels and build a UUID map:

```
query_data: fetch all labels for the workspace (not the team)
→ filter to those whose parent label name is "Type"
→ build a map: { "Bug": "<uuid>", "Feature": "<uuid>", ... }
```

If the workspace has no "Type" label group, surface that to the user before proceeding.

### Step 3 — Fetch and filter issues

Use `query_data` to list issues for the team, **including each issue's label IDs**.
Paginate (limit 50, use `cursor`) until done.

Filter client-side: keep only issues where none of the issue's label IDs appear in the
Step 2 UUID map values (i.e., no Type label assigned yet).

Announce the total candidate count before continuing. If there are more than 25 candidates,
process in chunks of 25, confirming each chunk before the next.

### Step 4 — Classify each issue

For each candidate issue, read its **title** and **description**. Apply heuristics in **priority
order** — stop at the first match:

1. Title or description is clearly about testing work: mentions "test", "tests", "coverage",
   "test infra", "test suite", "unit test", "integration test" → **Tests**
2. Title or description is clearly about documentation: mentions "docs", "README", "changelog",
   "RFC", "spec", "guide", "write-up", "develop.sentry.dev" → **Docs**
3. Title contains `[META]`, `[EPIC]`, "Track", "Tracking"; or description has a checklist of
   Linear/GitHub issue links → **Tracking Issue**
4. Title starts with "Fix", "Fixes", "Broken", "Error", "Crash", "Regression" → **Bug**
5. Title starts with "Improve", "Optimize", "Enhance"; or is a refactor with user-visible impact
   (changes behavior, performance, or API ergonomics) → **Improvement**
6. Title starts with "Investigate", "Spike", "Migrate", "Update", "Clean up", "Chore"; or is a
   refactor with no user-visible change → **Task**
7. Title starts with "Add", "Support", "Implement", "New", "Introduce"; or adds something that
   doesn't exist yet → **Feature**

When ambiguous between **Feature** and **Improvement**: does it add something that doesn't exist
at all, or make something existing better? Former → Feature, latter → Improvement.

When ambiguous between **Task** and **Improvement**: is there a user-visible benefit (better
performance, reliability, DX, API ergonomics)? If yes → Improvement. If it's purely internal
with no user-visible impact → Task.

Assign exactly one Type label. Mark confidence:
- **High** — clear signal in title or description
- **Low** — genuinely ambiguous; flag for human review, don't guess

### Step 5 — Present the plan

Show a summary table before writing anything:

```
| Issue   | Title                  | Proposed Type | Confidence          |
|---------|------------------------|---------------|---------------------|
| SDK-123 | Fix crash on startup   | Bug           | High                |
| SDK-456 | Add OAuth support      | Feature       | High                |
| SDK-789 | Some vague title       | Task          | Low — please review |
```

Ask: *"Should I apply these labels? Reply with any corrections (e.g. 'SDK-789 → Bug') and I'll
update the plan before applying."*

If the user provides corrections, update the table and show the revised plan before proceeding.

### Step 6 — Apply labels

For each approved classification:

1. Call `get_issue` to fetch the issue's **current** label IDs immediately before writing
2. Append the Type label UUID from the Step 2 map
3. Call `save_issue` with the full combined label set

> **Always re-fetch and always pass the full combined set.** Linear replaces labels on
> update — labels added between Step 3 and now would be silently stripped if you use
> the cached label list.

Report as you go: "SDK-123 → Bug", "SDK-456 → Feature".

## Edge Cases

- **Issue already has a Type label:** Skip it; don't overwrite human classifications.
- **Very short title, no description:** Flag as low-confidence; surface for human review.
- **Issue belongs to multiple types:** Pick the primary type (bug fix with a docs update → Bug).
