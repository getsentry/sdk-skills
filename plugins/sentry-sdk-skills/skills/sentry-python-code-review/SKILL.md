---
name: sentry-python-code-review
description: Additional code review checks specific to sentry-python. Use alongside the general code-review skill when reviewing PRs in sentry-python. Triggers on "code review", "review PR", "review changes" when working in sentry-python. Covers Sentry conventions conformance, public API test coverage, and streamed span test assertions.
allowed-tools: Read Grep Glob Bash WebFetch
---

# Sentry Python Code Review

Apply these checks **in addition to** the general `code-review` skill when reviewing code in the `sentry-python` repository.

## Step 1: Identify Changes

Find all changes in the current branch compared to the base branch:

```bash
git diff main...HEAD --name-only
```

Separate the changed files into:
- **Source files**: files under `sentry_sdk/`
- **Test files**: files under `tests/`

## Step 2: Sentry Conventions Conformance

Use `WebFetch` to fetch the current conventions from:

```
https://getsentry.github.io/sentry-conventions/attributes/
```

For each source file in the diff, look for span attributes, operation names, or semantic conventions being set. Common patterns to scan for:

- `op=` or `op:` assignments
- `set_data(`, `set_attribute(`
- String literals matching known attribute namespaces: `db.`, `http.`, `net.`, `messaging.`, `gen_ai.`, `cache.`, `queue.`, `sentry.`
- Constants referencing span data or operations (e.g., `SPANDATA.`, `OP.`)

For each attribute or convention found in the diff:

1. **Check if it exists** on the fetched conventions page
2. If it **does exist**: verify the usage matches the defined type and semantics
3. If it **does not exist**: flag it to the code author — the attribute must be added to [sentry-conventions](https://github.com/getsentry/sentry-conventions) before or alongside this PR

### Output for this check

```
### Conventions Conformance

#### Missing from sentry-conventions

1. **[file:line]** Attribute `my.custom.attribute` is not listed in sentry-conventions
   - Action: Add this attribute to https://github.com/getsentry/sentry-conventions before merging

#### Mismatched usage

1. **[file:line]** Attribute `db.system` expects type `string`, but is set as an integer
   - Convention: https://getsentry.github.io/sentry-conventions/attributes/
```

## Step 3: Public API Test Coverage

Scan all test files in the diff. Flag any test that **directly calls or asserts on internal functions** — those with names prefixed by `_` (single underscore).

Tests should exercise the public-facing SDK API that users interact with, such as:
- `sentry_sdk.init()`
- `sentry_sdk.capture_message()`, `sentry_sdk.capture_exception()`, `sentry_sdk.capture_event()`
- `sentry_sdk.start_span()`, `sentry_sdk.start_transaction()`
- Integration classes and their public methods
- Public configuration options and their effects

### What to flag

- Direct calls to `_`-prefixed functions (e.g., `_process_event()`, `_apply_tracing()`)
- Assertions on return values of `_`-prefixed functions
- Imports of `_`-prefixed functions from `sentry_sdk` internals

### What is acceptable

- Testing that internal behavior is triggered **through** the public API (e.g., calling `capture_message()` and asserting the resulting envelope contains the right data)
- Using `_`-prefixed helpers defined **within the test file itself** for test setup/utilities

### Output for this check

```
### Public API Test Coverage

1. **[test_file.py:line]** Test directly calls internal function `_serialize_span()`
   - Suggestion: Test this behavior through the public `start_span()` API instead

2. **[test_file.py:line]** Test imports `_apply_default_options` from `sentry_sdk._internals`
   - Suggestion: Assert the effect of options through `sentry_sdk.init()` and observable behavior
```

## Step 4: Streamed Span Test Assertions

In test files that deal with spans (look for `start_span`, `start_transaction`, span assertions, or envelope/transaction event inspection), verify the tests make **complete assertions** about streamed spans.

### Required assertions

Each span test should assert:

1. **Total span count**: The test verifies the exact number of spans produced (e.g., `assert len(spans) == 3`)
2. **Span order**: The test checks that spans appear in the expected sequence, not just that individual spans exist somewhere in the list
3. **Span correctness**: Each span's key properties are verified — at minimum `op` and `description`, plus relevant attributes

### What to flag

- Tests that only check `any(s["op"] == "expected" for s in spans)` without verifying count or order
- Tests that assert a span exists but don't verify its position relative to other spans
- Tests with no assertion on the total number of spans (allowing extra unexpected spans to go unnoticed)

### What is acceptable

- Tests for error/exception scenarios where span count may legitimately vary
- Tests that explicitly document why order is non-deterministic (e.g., concurrent async spans) with a comment explaining the rationale

### Output for this check

```
### Streamed Span Test Assertions

1. **[test_file.py:line]** Span test does not assert total span count
   - Found: `assert spans[0]["op"] == "db"` but no `assert len(spans) == ...`
   - Suggestion: Add `assert len(spans) == N` to catch unexpected extra spans

2. **[test_file.py:line]** Span test checks existence but not order
   - Found: `assert any(s["op"] == "http.client" for s in spans)`
   - Suggestion: Assert position explicitly, e.g., `assert spans[1]["op"] == "http.client"`
```

## Step 5: Report Findings

Present all findings in a single report, grouped by check:

```
## Sentry Python Code Review — Additional Checks

### 1. Conventions Conformance
(findings from Step 2)

### 2. Public API Test Coverage
(findings from Step 3)

### 3. Streamed Span Test Assertions
(findings from Step 4)

### Summary
- X convention issues found
- Y internal API test issues found
- Z span assertion issues found
```

If a check finds no issues, note it as passing:

```
### 2. Public API Test Coverage
✅ All tests exercise the public-facing API — no issues found.
```
