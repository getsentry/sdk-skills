---
name: feature-status
description: Check which Sentry SDKs have implemented a feature. Use when auditing SDK coverage, checking implementation progress, or planning alignment work.
allowed-tools: Read Bash Write
compatibility: Requires gh CLI with authentication.
---

# SDK Feature Status

Check implementation status of a feature across Sentry SDKs.

## Usage

User provides:
- Feature name or description
- Search pattern (class name, function name, config option, etc.)
- Optionally: specific SDKs to check

## SDK Repositories

| SDK | Repository |
|-----|------------|
| python | getsentry/sentry-python |
| javascript | getsentry/sentry-javascript |
| go | getsentry/sentry-go |
| java | getsentry/sentry-java |
| dotnet | getsentry/sentry-dotnet |
| ruby | getsentry/sentry-ruby |
| php | getsentry/sentry-php |
| rust | getsentry/sentry-rust |
| cocoa | getsentry/sentry-cocoa |
| android | getsentry/sentry-java |
| react-native | getsentry/sentry-react-native |
| flutter | getsentry/sentry-dart |
| elixir | getsentry/sentry-elixir |

## Process

### 1. Get Search Pattern

If user didn't provide a search pattern, ask:

> What code pattern should I search for? (e.g., class name, function name, config key)

### 2. Check Each SDK

For each SDK, run a single code search:

```bash
gh search code "<pattern>" --repo getsentry/<repo> --limit 1 --json path
```

If results found: mark as implemented.
If no results: mark as not implemented.

If rate limited, wait briefly and retry. If persistent failures, note and continue.

### 3. Output Results

Display a simple table:

```
Feature: <name>
Pattern: <search-pattern>

| SDK        | Status | File |
|------------|--------|------|
| python     | Yes    | sentry_sdk/integrations/feature.py |
| javascript | Yes    | packages/core/src/feature.ts |
| go         | No     | - |
...

Summary: 8/13 SDKs implemented
```

### 4. Save Results (Optional)

If user wants to save for later use:

```bash
mkdir -p .sdk-align
```

Write `.sdk-align/status.json`:

```json
{
  "feature": "<name>",
  "pattern": "<pattern>",
  "checked_at": "<timestamp>",
  "results": {
    "python": {"implemented": true, "file": "path/to/file.py"},
    "go": {"implemented": false}
  }
}
```

## Notes

- This is a quick scan, not exhaustive verification
- False negatives possible if feature uses different naming
- For deeper analysis, check individual repos manually
- Rate limits may slow down checks; be patient
