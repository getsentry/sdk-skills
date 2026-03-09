---
name: span-convention-review
description: Review OpenTelemetry tracing span changes in SDK repositories for conformance to Sentry Conventions and OTel Semantic Conventions. Use when reviewing span instrumentation, "review spans", "check span conventions", "span review", "OTel span check", "tracing convention review".
model: opus
allowed-tools: Read Grep Glob Bash WebFetch
---

# Span Convention Review

Review changes in SDK repositories that involve OpenTelemetry tracing spans to ensure they conform to Sentry Conventions and, where Sentry conventions are underspecified, to OpenTelemetry Semantic Conventions.

## Step 1: Identify Span Changes

Find all span-related changes in the current branch compared to the base branch:

```bash
git diff main...HEAD
```

Look for patterns indicating span creation or modification:
- `start_span`, `start_child`, `startSpan`, `StartSpan`
- `op=`, `op:`, setting span operation names
- `set_data`, `set_attribute`, `setAttribute`, `SetAttribute`
- `description=`, `name=`, span naming
- `SPANDATA.`, `SpanData.`, span data constants
- `OP.`, span operation constants
- References to `gen_ai.`, `db.`, `http.`, `cache.`, `queue.`, `messaging.`, `resource.`, `ui.`

If no diff is available, ask the user which files or changes to review.

## Step 2: Classify Spans by Module

Determine which Sentry Insights Module each span belongs to based on its operation prefix. Use this mapping to identify the relevant convention URLs:

| Op Prefix / Pattern | Sentry Module | Sentry Convention URL | OTel Semconv URL |
|---|---|---|---|
| `gen_ai.*` | AI Agents | https://develop.sentry.dev/sdk/telemetry/traces/modules/ai-agents/ | https://opentelemetry.io/docs/specs/semconv/gen-ai/ |
| `db.*`, `db.sql.*` | Queries | https://develop.sentry.dev/sdk/telemetry/traces/modules/queries/ | https://opentelemetry.io/docs/specs/semconv/database/ |
| `http.client` | Requests | https://develop.sentry.dev/sdk/telemetry/traces/modules/requests/ | https://opentelemetry.io/docs/specs/semconv/http/http-spans/ |
| `cache.*` | Caches | https://develop.sentry.dev/sdk/telemetry/traces/modules/caches/ | N/A |
| `queue.*` | Queues | https://develop.sentry.dev/sdk/telemetry/traces/modules/queues/ | https://opentelemetry.io/docs/specs/semconv/messaging/ |
| `resource.script`, `resource.css`, `resource.img` | Assets | https://develop.sentry.dev/sdk/telemetry/traces/modules/assets/ | N/A |
| `app.start.*` | App Starts | https://develop.sentry.dev/sdk/telemetry/traces/modules/app-starts/ | N/A |
| `ui.load*` | Screen Loads | https://develop.sentry.dev/sdk/telemetry/traces/modules/screen-loads/ | N/A |
| `ui.interaction.*` | Web Vitals | https://develop.sentry.dev/sdk/telemetry/traces/modules/web-vitals/ | N/A |

## Step 3: Fetch Current Conventions

For each module identified in Step 2, use `WebFetch` to retrieve the **Sentry Convention page** from the URL in the table above. Sentry conventions are the primary authority.

If the Sentry convention page is underspecified for a particular attribute, span name format, or operation pattern, also fetch the corresponding **OTel Semantic Conventions page** for supplementary guidance.

**Do not assume or hardcode convention details.** Always fetch the live documentation to ensure you are reviewing against the latest version.

## Step 4: Review Each Span

For each span change, verify conformance against the fetched conventions for its module. Check the following categories:

### 4a. Span Operation (`op`)
Verify the `op` value matches the expected pattern defined in the fetched Sentry convention for the module.

### 4b. Span Name / Description
Verify the span name follows the format specified in the fetched convention.

### 4c. Required Attributes
Verify all required span data attributes listed in the fetched convention are set.

### 4d. Optional/Recommended Attributes
Flag if commonly expected optional attributes (listed in the fetched convention) are missing. These are "should have" not blockers.

### 4e. Data Sensitivity
Check that spans do not leak sensitive data:
- Database queries must have user-supplied values scrubbed/parameterized
- AI spans: input/output messages and system instructions should only be set when explicitly opted in (e.g., via a "send default PII" or "send prompts" setting)
- HTTP spans should redact sensitive URL parameters
- Cache keys should not contain raw user identifiers unless expected

### 4f. Attribute Types
Verify attributes use correct types as specified in the conventions:
- Complex data (messages, tool definitions) must be stringified JSON, not raw objects
- Boolean attributes must be actual booleans
- Numeric attributes must be integers/numbers, not strings
- Array attributes must be arrays

## Step 5: Report Findings

Present findings grouped by severity:

### Required Fixes (convention violations)
Issues that MUST be fixed — missing required attributes, wrong op format, incorrect span name patterns.

### Recommended Improvements
Issues that SHOULD be addressed — missing recommended attributes, suboptimal naming.

### Notes
Informational observations — things that look correct, edge cases to be aware of, or areas where conventions are ambiguous.

For each finding, include:
1. **File and line** where the issue occurs
2. **What's wrong** — the specific convention being violated
3. **Expected** — what the convention requires
4. **Actual** — what the code currently does
5. **Convention source** — link to the specific Sentry or OTel convention page

### Output Format

```
## Span Convention Review

### Required Fixes

1. **[file:line]** Missing required attribute `gen_ai.operation.name`
   - Convention: All AI agent spans MUST set `gen_ai.operation.name`
   - Source: https://develop.sentry.dev/sdk/telemetry/traces/modules/ai-agents/

2. **[file:line]** Incorrect span name format
   - Expected: `"chat {model}"` (e.g., `"chat gpt-4"`)
   - Actual: `"OpenAI chat completion"`
   - Source: https://develop.sentry.dev/sdk/telemetry/traces/modules/ai-agents/

### Recommended Improvements

1. **[file:line]** Missing recommended attribute `gen_ai.response.finish_reasons`
   - This attribute helps with debugging incomplete responses
   - Source: https://develop.sentry.dev/sdk/telemetry/traces/modules/ai-agents/

### Notes

- All cache spans correctly implement the `cache.hit` attribute
- Queue span naming follows OTel convention (Sentry convention is underspecified here)
```

## Handling Edge Cases

- **Unknown module**: If a span op doesn't match any known module, flag it and check if there's a new or updated convention page. If no convention exists, fall back to general OTel semantic conventions.
- **Multiple modules**: Some integrations produce spans for multiple modules (e.g., Redis creates both `cache.*` and `db.redis` spans). Review each span type against its respective module convention.
- **SDK-specific patterns**: Different SDKs use different APIs to create spans (Python uses `start_span(op=..., name=...)`, JavaScript uses `startSpan({op: ..., name: ...})`, etc.). Focus on the semantic correctness of op, name, and attributes regardless of the API surface.
- **Origin attribute**: While not part of module conventions, verify that `origin` is set on auto-instrumented spans (pattern: `auto.{category}.{integration}`). This is a general Sentry SDK convention.
