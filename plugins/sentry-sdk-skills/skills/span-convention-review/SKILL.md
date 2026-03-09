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

Determine which Sentry Insights Module each span belongs to based on its operation prefix. Use this mapping:

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

For each module identified in Step 2, fetch the **Sentry Convention page** using `WebFetch` to get the latest conventions. Sentry conventions are the primary authority. Only consult the OTel Semconv page if the Sentry convention is underspecified for a particular attribute or pattern.

**Do not rely solely on the conventions embedded in this skill.** Always fetch the live documentation to ensure you are reviewing against the latest version. The reference sections below serve as a guide for what to look for.

## Step 4: Review Each Span

For each span change, verify conformance against the conventions for its module. Check the following categories:

### 4a. Span Operation (`op`)

Verify the `op` value matches the expected pattern for the module:

- **AI Agents**: `gen_ai.{operation_name}` where operation is one of: `chat`, `embeddings`, `generate_content`, `text_completion`, `create_agent`, `invoke_agent`, `execute_tool`, `handoff`
- **Queries**: Must be prefixed with `db` — accepted: `db`, `db.query`, `db.sql.query`. Note: `db.redis`, `db.sql.room` are NOT indexed by the Queries module
- **Requests**: Must be exactly `http.client`
- **Caches**: One of `cache.get`, `cache.put`, `cache.remove`, `cache.flush`
- **Queues**: One of `queue.publish`, `queue.create`, `queue.receive`, `queue.process`, `queue.settle`
- **Assets**: One of `resource.script`, `resource.css`, `resource.img`
- **App Starts**: `app.start.cold` or `app.start.warm`
- **Screen Loads**: `ui.load`, `ui.load.initial_display`, `ui.load.full_display`
- **Web Vitals**: `ui.interaction.click`, `ui.interaction.press`, `ui.interaction.hover`, `ui.interaction.drag`

### 4b. Span Name / Description

Verify the span name follows the expected format:

- **AI Agents**:
  - `create_agent`: `"create_agent {agent_name}"`
  - `invoke_agent`: `"invoke_agent {agent_name}"` or `"invoke_agent {call_id}"` if unnamed
  - AI client (chat/embeddings/etc): `"{operation_name} {model}"` (e.g., `"chat o3-mini"`)
  - `execute_tool`: `"execute_tool {tool_name}"`
  - `handoff`: `"handoff from {from_agent} to {to_agent}"`
- **Queries**: The full scrubbed query with user values replaced by placeholders (`?`, `%s`, `:c0`, `$1`). MongoDB: values replaced with `"?"`
- **Requests**: `"{HTTP_METHOD} {URL}"` (e.g., `"GET https://example.com/data.json"`)
- **Caches**: Cache key(s) separated by commas (e.g., `"posts"` or `"article1, article2"`)
- **Queues**: No specific name convention in Sentry; OTel recommends `"{operation_name} {destination}"`
- **Assets**: Full URL of the asset including query parameters

### 4c. Required Attributes

Verify all required span data attributes are set:

**AI Agents** (all spans require `gen_ai.operation.name`):
- AI client spans: `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.response.model`
- Agent spans: `gen_ai.operation.name`; should set `gen_ai.agent.name`, `gen_ai.request.model`, `gen_ai.pipeline.name`
- Tool spans: `gen_ai.operation.name`; should set `gen_ai.tool.name`
- Token usage: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.total_tokens`

**Queries**:
- `db.system` (required minimum)
- Recommended: `db.operation`, `db.name`, `server.address`, `server.port`

**Requests**:
- `server.address` (especially when description has relative URLs)
- `http.response.status_code`

**Caches**:
- `cache.hit` (boolean, required for read operations)
- `cache.key` (array)

**Queues**:
- `messaging.message.id`
- `messaging.destination.name`
- `messaging.system`

**Assets**:
- `http.response_content_length`

### 4d. Optional/Recommended Attributes

Flag if commonly expected optional attributes are missing. These are "should have" not blockers:

**AI Agents**: `gen_ai.system`, `gen_ai.request.temperature`, `gen_ai.request.max_tokens`, `gen_ai.response.finish_reasons`
**Queries**: `db.operation`, `db.name`, `server.address`, `server.port`
**Requests**: `http.query`, `http.request_method`, `http.fragment`, `server.port`
**Caches**: `cache.item_size`, `cache.ttl`, `network.peer.address`, `network.peer.port`
**Queues**: `messaging.message.body.size`, `messaging.message.receive.latency`
**Assets**: `http.decoded_response_content_length`, `http.response_transfer_size`, `resource.render_blocking_status`

### 4e. Data Sensitivity

Check that spans do not leak sensitive data:
- Database queries must have user-supplied values scrubbed/parameterized
- AI spans: `gen_ai.input.messages`, `gen_ai.output.messages`, and `gen_ai.system_instructions` should only be set when explicitly opted in (e.g., via a "send default PII" or "send prompts" setting)
- HTTP spans should redact sensitive URL parameters or use `SENSITIVE_DATA_SUBSTITUTE` patterns
- Cache keys should not contain raw user identifiers unless expected

### 4f. Attribute Types

Verify attributes use correct types:
- Complex data (messages, tool definitions) must be stringified JSON, not raw objects
- Boolean attributes (`cache.hit`, `gen_ai.response.streaming`) must be actual booleans
- Numeric attributes (token counts, status codes, ports) must be integers/numbers, not strings
- Array attributes (`cache.key`, `gen_ai.response.finish_reasons`) must be arrays

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
