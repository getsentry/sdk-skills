---
name: linear-sdk-telemetry-labeler
description: >
  Classify and apply SDK Telemetry labels to Linear issues based on their title and description.
  Use when triaging issues related to SDK telemetry signals: errors, traces, spans, profiles,
  replays, logs, metrics, cron checks, client reports, user feedbacks, or attachments.
  Triggers: "sdk telemetry label", "telemetry labeler", "label telemetry issues", "tag telemetry".
allowed-tools: mcp__linear-server__query_data mcp__linear-server__get_issue mcp__linear-server__save_issue AskUserQuestion
compatibility: Requires the Linear MCP server to be configured.
---

# Linear SDK Telemetry Labeler

Classifies Linear issues and applies an **SDK Telemetry** label from the Sentry workspace's label
taxonomy based on the content of each issue's title and description.

## Requirements

This skill requires the [Linear MCP server](https://github.com/linear/linear-mcp) to be configured.

## SDK Telemetry Label Reference

SDK Telemetry labels in the Sentry workspace (workspace-level, not per-team):

| Label | When to use |
|---|---|
| **Errors** | Error events, exception capture, crash reporting, error grouping, `captureException`, `captureMessage`, error payloads, fingerprinting |
| **Traces** | Distributed tracing, trace context propagation, trace sampling, `sentry-trace` header, W3C `traceparent`/baggage |
| **Spans** | Span creation, span attributes/tags, auto-instrumentation producing spans, `startSpan`, span operations, OTel span APIs |
| **Profiles** | Profiling, continuous profiling, function-level profiling, profiler lifecycle, profile envelope item |
| **Replays** | Session replay, replay recording/capture, `replayId`, rrweb integration, replay worker, replay envelope item |
| **Logs** | Structured log capture, `sentry.logger`, `logger.info/warn/error`, log envelope item |
| **Metrics** | Custom metrics, counters, gauges, histograms, distributions, `metrics.increment`, `metrics.gauge`, `metrics.distribution` |
| **Checks** | Cron monitors, check-ins, `captureCheckIn`, monitor slugs, scheduled task tracking |
| **Client Reports** | Client reports, discarded events reporting, envelope items for client reports, `sendClientReports` |
| **User Feedbacks** | User feedback widget, feedback form, `captureFeedback`, crash reporter UI, `showReportDialog` |
| **Attachments** | Attachments, file uploads, `addAttachment`, screenshot capture, view hierarchy dumps |

## Workflow

### Step 1 — Identify issues to process

Two modes:

**A — List of Linear IDs provided** (e.g. `SDK-123, SDK-456`):
- Fetch each issue by ID using `get_issue`. Skip the team-fetch step entirely.

**B — No IDs provided**:
- Infer the team: (1) current GitHub repository name — for the `getsentry` org the repo name
  matches the Linear team name, (2) explicitly stated by the user, (3) if still ambiguous, ask.
- Use `query_data` to list issues for the team, **including each issue's label IDs**.
  Paginate (limit 50, use `cursor`) until done.

### Step 2 — Fetch label IDs

Use `query_data` to fetch workspace-level labels and build a UUID map:

```
query_data: fetch all labels whose parent label name is "SDK Telemetry"
→ build a map: { "Errors": "<uuid>", "Spans": "<uuid>", ... }
```

If the workspace has no "SDK Telemetry" label group, surface that to the user before proceeding.

### Step 3 — Filter candidates

Filter issues client-side: keep only issues where none of the issue's label IDs appear in the
Step 2 UUID map values (i.e., no SDK Telemetry label assigned yet).

Announce the total candidate count before continuing. If there are more than 25 candidates,
process in chunks of 25, confirming each chunk before the next.

### Step 4 — Classify each issue

For each candidate issue, read its **title** and **description**. Apply heuristics in **priority
order** — stop at the first match:

1. Title or description mentions "replay", "session replay", "rrweb", "replay recording",
   "replay capture" → **Replays**
2. Title or description mentions "profile", "profiling", "profiler", "continuous profil" → **Profiles**
3. Title or description mentions "cron", "check-in", "checkin", "captureCheckIn", "monitor slug",
   "scheduled task" → **Checks**
4. Title or description mentions "client report", "discarded event", "sendClientReports" → **Client Reports**
5. Title or description mentions "feedback", "showReportDialog", "captureFeedback",
   "feedback widget", "crash reporter UI" → **User Feedbacks**
6. Title or description mentions "attachment", "addAttachment", "screenshot capture",
   "view hierarchy" → **Attachments**
7. Title or description mentions "metric", "counter", "gauge", "histogram", "distribution",
   "metrics.increment", "metrics.gauge" → **Metrics**
8. Title or description mentions "log", "logger", "structured log", "sentry.logger",
   "log envelope" → **Logs**
9. Title or description mentions "span", "startSpan", "auto-instrument", "OTel span",
   "span attribute", "span operation" → **Spans**
10. Title or description mentions "trace", "tracing", "traceparent", "baggage", "sentry-trace",
    "distributed trac", "trace propagat", "trace sampl" → **Traces**
11. Title or description mentions "error", "exception", "crash", "captureException",
    "captureMessage", "error group", "fingerprint" → **Errors**

When ambiguous between **Spans** and **Traces**: spans are individual units of work; traces are
the propagation/sampling/context layer. An issue about instrumenting a library → Spans. An issue
about `sentry-trace` header or sampling → Traces.

When ambiguous between **Logs** and **Errors**: structured log capture → Logs. Error/exception
capture → Errors.

Assign exactly one SDK Telemetry label. Mark confidence:
- **High** — clear signal in title or description
- **Low** — genuinely ambiguous; flag for human review, don't guess

### Step 5 — Present the plan

Show a summary table before writing anything:

```
| Issue   | Title                          | Proposed Label | Confidence          |
|---------|--------------------------------|----------------|---------------------|
| SDK-123 | Add span for DB queries        | Spans          | High                |
| SDK-456 | Fix trace propagation in Fetch | Traces         | High                |
| SDK-789 | Improve SDK startup            | —              | Low — please review |
```

Ask: *"Should I apply these labels? Reply with any corrections (e.g. 'SDK-789 → Logs') and I'll
update the plan before applying."*

If the user provides corrections, update the table and show the revised plan before proceeding.

### Step 6 — Apply labels

For each approved classification:

1. Call `get_issue` to fetch the issue's **current** label IDs immediately before writing
2. Append the SDK Telemetry label UUID from the Step 2 map
3. Call `save_issue` with the full combined label set

> **Always re-fetch and always pass the full combined set.** Linear replaces labels on
> update — labels added between Step 3 and now would be silently stripped if you use
> the cached label list.

Report as you go: "SDK-123 → Spans", "SDK-456 → Traces".

## Edge Cases

- **Issue already has an SDK Telemetry label:** Skip it; don't overwrite human classifications.
- **Issue could belong to multiple signals:** Pick the most specific one (e.g., an issue about
  replay span instrumentation → Replays, not Spans).
- **Very short title, no description:** Flag as low-confidence; surface for human review.
- **No clear telemetry signal:** Do not apply any label; flag for human review.
