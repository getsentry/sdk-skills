---
name: sdk-feature-status
description: Check SDK feature implementation status across all Sentry SDKs. Analyzes develop docs, finds reference implementations, and reports which SDKs have implemented, need updates, or are missing the feature. Creates shared context for other sdk-* skills.
model: sonnet
allowed-tools: Read Grep Glob Bash Write AskUserQuestion TodoWrite
compatibility: Requires gh CLI (GitHub CLI) with authentication configured.
---

# SDK Feature Status Checker

Check implementation status of a feature across all Sentry SDKs based on develop docs and reference implementations.

## When to Use This Skill

Use this skill when:
- You want to audit which SDKs have implemented a feature
- You're starting SDK alignment work and need to understand the landscape
- You need to find the reference implementation for a feature
- You want to create shared context for other `sdk-*` skills

## Shared Context System

This skill creates/updates `.sdk-align/` directory with shared context:

```
.sdk-align/
├── context.json          # Feature context (develop doc, reference impl, Linear issue)
└── status-report.json    # Implementation status per SDK
```

Other `sdk-*` skills can read this context to work together seamlessly.

## SDK to Repository Mapping

- `python` → `getsentry/sentry-python`
- `javascript` → `getsentry/sentry-javascript`
- `ruby` → `getsentry/sentry-ruby`
- `php` → `getsentry/sentry-php`
- `go` → `getsentry/sentry-go`
- `java` → `getsentry/sentry-java`
- `dotnet` → `getsentry/sentry-dotnet`
- `rust` → `getsentry/sentry-rust`
- `android` → `getsentry/sentry-java`
- `cocoa` → `getsentry/sentry-cocoa`
- `react-native` → `getsentry/sentry-react-native`
- `flutter` → `getsentry/sentry-dart`
- `unity` → `getsentry/sentry-unity`
- `unreal` → `getsentry/sentry-unreal`
- `native` → `getsentry/sentry-native`
- `elixir` → `getsentry/sentry-elixir`
- `perl` → `getsentry/sentry-perl`
- `clojure` → `getsentry/sentry-clj`
- `kotlin` → `getsentry/sentry-kotlin-multiplatform`

## Process

### Step 1: Gather Develop Doc

Ask user for the develop doc. Accept:
- **PR URL**: `https://github.com/getsentry/sentry-docs/pull/XXXX` or `https://github.com/getsentry/develop/pull/XXXX`
- **Commit SHA**: From either sentry-docs or develop repo
- **Direct URL**: Link to develop.sentry.dev
- **File path**: Local path to develop doc MDX file

Fetch the develop doc:

```bash
# If PR URL provided
gh pr view <PR_NUM> --repo <REPO> --json title,body,files,state,url

# If commit SHA provided
gh api repos/<REPO>/commits/<SHA>
```

Analyze the develop doc to extract:
- Feature name
- Alignment scope (Core/Domain/Platform)
- Key requirements (MUST/SHOULD/MAY)
- Configuration options
- Public API details

### Step 2: Find Reference Implementation

**If user provides reference implementation:**
- SDK name and optional PR URL
- Fetch PR details

**If not provided, search for it:**

1. Search for PRs mentioning the develop doc PR number:
```bash
gh search prs --match body "<PR_NUMBER>" --owner getsentry --limit 10
```

2. Search for PRs with related titles:
```bash
gh search prs --match title "<feature-name>" --owner getsentry --state merged --limit 10
```

3. Search code for feature-specific identifiers:
```bash
gh search code "<config-option-name>" --owner getsentry --limit 10
```

4. Filter results to SDK repos only (exclude sentry-docs, sentry, etc.)

5. Find the earliest merged PR as reference implementation

**Extract reference implementation details:**
- SDK name
- PR URL and merge date
- Key code patterns (class names, function names, config options)
- File structure
- Test patterns

### Step 3: Check All SDKs

For each SDK in the mapping, check implementation status using a **three-stage detection approach**:

#### Stage 1: Search for Develop Doc References

```bash
# Search PRs mentioning develop doc
gh search prs --repo <SDK_REPO> --match body "<develop-doc-url>" --limit 5

# Search commits mentioning develop doc
gh search commits --repo <SDK_REPO> "<develop-doc-title>" --limit 5
```

#### Stage 2: Search Git History

```bash
# Search for PRs with related titles
gh search prs --repo <SDK_REPO> --match title "<feature-name>" --limit 5
gh search prs --repo <SDK_REPO> --match title "<config-option>" --limit 5

# Check PR states
for each found PR:
  gh pr view <PR_NUM> --repo <SDK_REPO> --json state,mergedAt,url
```

#### Stage 3: Search Code Patterns

```bash
# Search for configuration options from develop doc
gh search code "<config-option-name>" --repo <SDK_REPO>

# Search for class/function names from reference implementation
gh search code "<class-name>" --repo <SDK_REPO>
gh search code "<function-name>" --repo <SDK_REPO>
```

#### Categorize Status

For each SDK, determine one of:
- ✅ **Implemented** - Feature fully implemented, PR merged
- ⚠️ **Needs Review** - Partial implementation or open PR
- ❌ **Not Implemented** - Feature missing entirely
- 🚫 **Not Applicable** - Feature doesn't apply to this SDK (based on alignment scope)

Collect metadata:
- PR URL (if exists)
- Merge date (if merged)
- Implementation notes
- Deviation from spec (if detected)

### Step 4: Smart Filtering by Alignment Scope

Based on the develop doc's alignment scope, filter applicable SDKs:

**Core Scope:**
- All SDKs with distributed tracing support (all except maybe perl, clojure if unmaintained)

**Domain Scope:**
- **Backend**: python, ruby, php, go, java, dotnet, elixir, rust, perl, clojure
- **Frontend**: javascript (browser), react-native
- **Mobile**: android, cocoa, flutter, react-native
- **Universal**: Features that apply across domains

**Platform Scope:**
- Feature may vary by platform
- Check if spec defines platform-specific requirements

Mark SDKs as "Not Applicable" if they don't match the scope.

### Step 5: Generate Status Report

Create a formatted status table:

```
SDK Feature Status Report
=========================
Feature: <Feature Name>
Develop Doc: <URL>
Reference: <SDK> (<PR URL>)
Alignment Scope: <Core|Domain|Platform>

Implementation Status:
─────────────────────────────────────────────────────────────
SDK              Status              Details
─────────────────────────────────────────────────────────────
javascript       ✅ Implemented      PR #16313 (merged 2025-07-25)
python           ✅ Implemented      PR #5178 (merged 2025-12-01)
php              ⚠️  Needs Review     PR #1875 (open)
ruby             ❌ Not Implemented  No implementation found
go               ❌ Not Implemented  No implementation found
java             ❌ Not Implemented  No implementation found
dotnet           ❌ Not Implemented  No implementation found
...
─────────────────────────────────────────────────────────────

Summary:
  ✅ Implemented: 2 SDKs
  ⚠️  Needs Review: 1 SDK
  ❌ Not Implemented: 14 SDKs
  🚫 Not Applicable: 2 SDKs

SDKs needing implementation (14):
  ruby, go, java, dotnet, rust, android, cocoa, react-native,
  flutter, unity, unreal, native, elixir, kotlin
```

Display this table to the user.

### Step 6: Write Shared Context

Create `.sdk-align/` directory and write context files:

**`.sdk-align/context.json`:**
```json
{
  "version": "1.0",
  "feature": {
    "name": "Strict Trace Propagation",
    "developDoc": {
      "url": "https://github.com/getsentry/sentry-docs/pull/11912",
      "commit": "399f5292fdf53a0b78f7de01978c7b329d2ab717",
      "pr": 11912
    },
    "referenceImplementation": {
      "sdk": "javascript",
      "pr": "https://github.com/getsentry/sentry-javascript/pull/16313",
      "prNumber": 16313,
      "mergedAt": "2025-07-25T11:32:54Z"
    },
    "alignmentScope": "Core",
    "configOptions": ["strictTraceContinuation", "org"],
    "linearIssue": null
  },
  "generatedAt": "2026-01-17T23:45:00Z",
  "generatedBy": "sdk-feature-status"
}
```

**`.sdk-align/status-report.json`:**
```json
{
  "version": "1.0",
  "feature": "Strict Trace Propagation",
  "generatedAt": "2026-01-17T23:45:00Z",
  "summary": {
    "total": 19,
    "implemented": 2,
    "needsReview": 1,
    "notImplemented": 14,
    "notApplicable": 2
  },
  "sdks": {
    "javascript": {
      "status": "implemented",
      "pr": "https://github.com/getsentry/sentry-javascript/pull/16313",
      "prNumber": 16313,
      "mergedAt": "2025-07-25T11:32:54Z",
      "isReference": true
    },
    "python": {
      "status": "implemented",
      "pr": "https://github.com/getsentry/sentry-python/pull/5178",
      "prNumber": 5178,
      "mergedAt": "2025-12-01T16:08:16Z"
    },
    "php": {
      "status": "needs_review",
      "pr": "https://github.com/getsentry/sentry-php/pull/1875",
      "prNumber": 1875,
      "state": "open",
      "notes": "PR open since 2025-08-11"
    },
    "go": {
      "status": "not_implemented",
      "needsImplementation": true
    }
  }
}
```

Inform user that context has been saved for other `sdk-*` skills.

## Output

Display to user:
1. Formatted status table
2. Summary statistics
3. List of SDKs needing implementation
4. Location of saved context files
5. Suggested next steps (e.g., "Run /sdk-feature-generate go to implement for Go SDK")

## Guidelines

### Progress Tracking

**Use TodoWrite to track the status check process:**

Create a todo list with the following steps:
1. Gather develop doc
2. Find reference implementation
3. Check all SDKs for implementation status
4. Generate status report
5. Write shared context files

Mark each step as in_progress when starting and completed when finished. When checking SDKs, consider adding individual todos for each SDK or SDK group to track progress through the ~19 SDK repositories.

### Handling Edge Cases

**Develop doc not accessible:**
- Ask user to provide develop doc content or file path
- If commit SHA, fetch raw file content via GitHub API

**Reference implementation not found:**
- Ask user which SDK should be reference
- Or continue without reference (use only develop doc)

**SDK repo not accessible:**
- Check `gh auth status`
- Skip that SDK and mark as "unknown"
- Continue with accessible SDKs

**Multiple implementations found:**
- Choose the earliest merged PR as reference
- List other implementations in notes

**Conflicting implementations:**
- Flag SDKs that have different implementations
- Note in status report for manual review

### Performance Optimization

- Run SDK checks in parallel where possible (use background processes)
- Cache GitHub API responses
- Limit search results to recent PRs (last 2 years)
- Skip code search for SDKs already found via PR/commit search

## Example Workflow

```
User: /sdk-feature-status https://github.com/getsentry/sentry-docs/pull/11912

Skill:
1. Fetches PR #11912 from sentry-docs
2. Extracts: "Strict Trace Propagation", Core scope
3. Searches for reference implementation
4. Finds JavaScript SDK PR #16313 (earliest)
5. Checks all 19 SDKs for implementation
6. Generates status report
7. Writes .sdk-align/context.json and status-report.json
8. Displays table and summary

User sees:
- 2 SDKs implemented
- 1 SDK needs review
- 14 SDKs not implemented
- Context saved for next steps
```

## References

- [Sentry SDK Philosophy](https://develop.sentry.dev/sdk/philosophy/)
- [GitHub CLI Documentation](https://cli.github.com/manual/)
