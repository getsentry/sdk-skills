---
name: sdk-feature-status
description: Use this skill when the user asks about SDK feature implementation status, SDK parity, or cross-SDK coverage. Triggers include: develop doc URLs (github.com/getsentry/sentry-docs/...), questions like 'which SDKs have X?', 'does Python SDK have X?', 'SDK parity for feature Y', or requests to check feature rollout status. Note: Even single-SDK queries will check all SDKs to provide parity context. Do NOT use for: SDK usage questions (how to use a feature), bug reports, SDK configuration help, or runtime behavior questions.
allowed-tools: Bash Task
---

# SDK Feature Status Checker

Check which Sentry SDKs have implemented a feature.

## Quick Reference

| Input | Action |
|-------|--------|
| Docs PR URL | Extract feature from PR title, find reference PR, check all SDKs |
| Develop doc URL | Extract feature from page title/URL, find reference PR, check all SDKs |
| Doc URL with anchor | Extract feature from anchor (e.g., `#feature-flags`), find reference PR, check all SDKs |
| Feature question | Extract keywords, search for reference implementation, check all SDKs |

| Output | Location |
|--------|----------|
| Formatted report | Displayed inline |

**Coverage:** All SDKs listed in the SDK table below
**Requires:** GitHub CLI (`gh`) with authentication

## ⚠️ Critical Requirements

- **Authentication required**: Run `gh auth status` before using. Unauthenticated requests are limited to 60/hour (vs 5,000/hour authenticated).
- **Rate limiting**: If you hit rate limits, the skill will return partial results. Wait 60 minutes or check `gh auth status`.
- **Network access**: Requires internet access to GitHub API.

## Usage

**Input:** Develop doc URL or free-form question
**Output:** Formatted table with status summary
**Time:** 2-5 minutes

**Examples:**
```bash
# Docs PR
/sdk-feature-status https://github.com/getsentry/sentry-docs/pull/16342

# Develop doc URL
/sdk-feature-status https://develop.sentry.dev/sdk/telemetry/replays/

# Develop doc with anchor (for pages with multiple features)
/sdk-feature-status https://develop.sentry.dev/sdk/expected-features/#feature-flags

# Free-form question
/sdk-feature-status Which SDKs have continuous profiling?
```

## SDK List

All SDKs in this table will be checked:

| SDK | Repository | Type | Path Filter | Notes |
|-----|------------|------|-------------|-------|
| `javascript` | `getsentry/sentry-javascript` | Frontend | - | Browser + Node.js environments |
| `electron` | `getsentry/sentry-electron` | Desktop | - | Desktop applications |
| `python` | `getsentry/sentry-python` | Backend | - | Full feature support |
| `java` | `getsentry/sentry-java` | Backend | `path:sentry/ -path:sentry-android/` | Shares repo with Android |
| `ruby` | `getsentry/sentry-ruby` | Backend | - | Full feature support |
| `php` | `getsentry/sentry-php` | Backend | - | Full feature support |
| `go` | `getsentry/sentry-go` | Backend | - | Full feature support |
| `dotnet` | `getsentry/sentry-dotnet` | Backend | - | Full feature support |
| `rust` | `getsentry/sentry-rust` | Backend | - | Full feature support |
| `elixir` | `getsentry/sentry-elixir` | Backend | - | Community-maintained |
| `laravel` | `getsentry/sentry-laravel` | Framework | - | PHP framework integration |
| `symfony` | `getsentry/sentry-symfony` | Framework | - | PHP framework integration |
| `cocoa` | `getsentry/sentry-cocoa` | Mobile | - | iOS/macOS, profiling supported |
| `android` | `getsentry/sentry-java` | Mobile | `path:sentry-android/` | Shares repo with Java, profiling supported |
| `react-native` | `getsentry/sentry-react-native` | Mobile | - | Wraps JavaScript + native SDKs |
| `flutter` | `getsentry/sentry-dart` | Mobile | - | Cross-platform mobile |
| `kotlin` | `getsentry/sentry-kotlin-multiplatform` | Mobile | - | Multiplatform support |
| `cordova` | `getsentry/sentry-cordova` | Mobile | - | Apache Cordova wrapper |
| `capacitor` | `getsentry/sentry-capacitor` | Mobile | - | Ionic Capacitor wrapper |
| `unity` | `getsentry/sentry-unity` | Gaming | - | Game engine specific |
| `godot` | `getsentry/sentry-godot` | Gaming | - | Game engine specific |
| `unreal` | `getsentry/sentry-unreal` | Gaming | - | Game engine specific, C++ based |
| `native` | `getsentry/sentry-native` | Native | - | C/C++, limited feature set |

**Notes:**
- Path filters are required for SDKs sharing repositories (Java/Android share `getsentry/sentry-java`)
- The table above is the source of truth for which SDKs are checked
- Notes column provides capability hints for "Not Applicable" status decisions

## Implementation

### Step 1: Parse Input

**If URL, extract feature name:**

| URL Type | Example | Extraction |
|----------|---------|------------|
| Docs PR | `github.com/getsentry/sentry-docs/pull/12345` | Feature name from PR title |
| Develop doc | `develop.sentry.dev/sdk/profiling/` | "profiling" from path |
| Doc with anchor | `develop.sentry.dev/sdk/features/#session-replay` | "session-replay" from anchor |

**If question, extract keywords:**
- Example: "Which SDKs have continuous profiling?" → keywords: `["continuous profiling"]`

**Note:** Anchors are critical for pages documenting multiple features. Using `#session-replay` targets that specific feature instead of the entire page.

### Step 2: Find Reference Implementation and Extract Patterns

This step is critical for accuracy. Extract search patterns from a reference implementation to find the feature across all SDKs.

#### 2.1 Find Reference PR

Search for earliest merged SDK PR:
```bash
# By develop doc (if PR link is in the doc)
gh search prs "sentry-docs/pull/12345" org:getsentry --json url,repository,number,mergedAt

# By keywords (search merged PRs only)
gh search prs "client reports is:merged" org:getsentry --limit 20 --json url,repository,number,title,mergedAt
```

Filter results:
- **Include**: SDK repos from the SDK List table
- **Exclude**: `getsentry/sentry`, `getsentry/sentry-docs`, `getsentry/develop`
- **Sort**: By `mergedAt` date (earliest first)
- **Select**: First result as reference implementation

If no results found, skip to Step 3 using only the keywords from Step 1.

#### 2.2 Fetch PR Details

Get full PR information:
```bash
gh pr view {pr_number} --repo {repo} --json title,body,files
```

This returns:
- `title`: PR title (e.g., "Add Client Reports to Python SDK")
- `body`: PR description with code examples
- `files`: List of changed files with paths

#### 2.3 Extract Patterns

Use the following algorithm to extract search patterns from the reference PR.

**A. Extract Feature Name from Title**

```
Input: "feat(python): Add Client Reports support"
Steps:
  1. Remove prefixes: "feat:", "Add ", "Implement ", "feature:", "fix:"
  2. Remove SDK references: "to Python SDK", "for iOS", "(python)"
  3. Remove suffixes: " support", " feature"
  4. Result: "Client Reports"
```

Common patterns:
- "Add {feature}" → {feature}
- "Implement {feature} for {SDK}" → {feature}
- "feat({sdk}): {feature}" → {feature}

**B. Extract Code Patterns from Body**

Parse the PR body markdown:

1. **Find code blocks**: Look for fenced code blocks (triple backticks) and inline code (single backticks)

2. **Extract class/interface names**:
   - Pattern: `[A-Z][a-zA-Z0-9]+` (CamelCase starting with capital)
   - Examples: `ClientReportManager`, `IClientReporter`, `ClientReportRecorder`
   - Filter out common words: `String`, `Boolean`, `Integer`, `Object`, `Array`

3. **Extract function/method names**:
   - Pattern: `[a-z][a-zA-Z0-9]+\(` (identifier followed by opening paren)
   - Examples: `sendClientReport(`, `recordLostEvent(`, `flushReports(`

4. **Extract configuration keys**:
   - Look for JSON/YAML structures: `"key": value` or `key: value`
   - Look for dict/object literals: `{...}`
   - Common config patterns: `enable*`, `send*`, `*Enabled`, `*Reports`
   - Examples: `sendClientReports`, `enableClientReports`, `client_reports_enabled`

**C. Extract Patterns from File Paths**

From the `files` field, examine file paths for naming clues:

```bash
# Example file paths:
sentry/client_reports.py          → "client_reports"
Sources/ClientReportManager.swift → "ClientReportManager"
client-report-manager.ts          → "client-report-manager"
```

Extraction rules:
- Take filename without extension
- Prefer newly added files over modified files
- Skip test files, docs, and config files
- Extract last path component (e.g., `src/core/reports.ts` → `reports`)

**D. Generate Language Variants**

Given a base pattern (e.g., `ClientReport`), generate all naming convention variants:

| Convention | Base | With Suffix |
|------------|------|-------------|
| CamelCase | `ClientReport` | `ClientReportManager`, `ClientReports` |
| camelCase | `clientReport` | `clientReportManager`, `clientReports` |
| snake_case | `client_report` | `client_report_manager`, `client_reports` |
| kebab-case | `client-report` | `client-report-manager`, `client-reports` |

Common suffixes: `Manager`, `Recorder`, `Handler`, `Service`, `Reporter`, `s` (plural)

**E. Categorize into Pattern Groups**

Organize extracted patterns into three categories:

```json
{
  "keywords": [
    "client reports",
    "client reporting",
    "client report"
  ],
  "code_patterns": [
    "ClientReport",
    "ClientReportManager",
    "ClientReportRecorder",
    "client_report",
    "client_report_manager"
  ],
  "config_options": [
    "sendClientReports",
    "send_client_reports",
    "enableClientReports",
    "client_reports_enabled"
  ]
}
```

Deduplication: Remove exact duplicates, keep case variations.

**F. Fallback Strategy**

If extraction yields fewer than 3 total patterns:

1. Use feature name from Step 1 as primary keyword
2. Generate variants:
   - Original: "session replay"
   - CamelCase: "SessionReplay"
   - snake_case: "session_replay"
   - kebab-case: "session-replay"
3. Add to keywords list
4. Continue to Step 3

**G. Pattern Selection for Search**

Use top patterns by frequency:
- Top 5 most common code_patterns
- Top 3 most common config_options
- All keywords (typically 1-3)

This prevents search query overload while maintaining coverage.

### Step 3: Check All SDKs with Streaming Output

Launch subagents in parallel batches while displaying results incrementally as they complete.

#### 3.1 Display Initial Header

Show header immediately before launching subagents:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SDK Feature Status: {feature_name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reference: {sdk} ({repo}#{pr}, merged {date})
Develop Doc: {url}

Checking {total} SDKs...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

This provides immediate feedback and context while subagents execute.

#### 3.2 Launch Subagents in Batches

Execute in 2-3 batches to balance parallelism vs API rate limits:

- **Batch 1** (10 SDKs): `javascript`, `python`, `java`, `android`, `ruby`, `php`, `go`, `dotnet`, `cocoa`, `react-native`
- **Batch 2** (10 SDKs): `rust`, `flutter`, `unity`, `electron`, `elixir`, `laravel`, `symfony`, `kotlin`, `native`, `unreal`
- **Batch 3** (3 SDKs): `cordova`, `capacitor`, `godot`

For each SDK, launch a subagent with these inputs:
- SDK name, repository, and path_filter (from SDK List table)
- Search patterns from Step 2 (keywords, code_patterns, config_options)

See `agents/sdk-checker.md` for complete subagent procedure and output schema.

#### 3.3 Display Results as They Complete

**Critical: Stream results immediately, don't wait for batch completion.**

As each subagent returns a result, display it in the appropriate category:

**For status = "implemented":**
```
✅ python  #1234  2024-01-15  getsentry/sentry-python
```

**For status = "needs_review":**
```
⚠️  java  #5678  (open)  getsentry/sentry-java
```

**For status = "not_implemented":**
```
❌ flutter  getsentry/sentry-dart
```

**For status = "not_applicable":**
```
🚫 native  getsentry/sentry-native  (Limited feature set)
```

**For status = "error":**
```
⚠️  rust  getsentry/sentry-rust  (Rate limited)
```

**Progress indicator**: Optionally show `[3/23]`, `[4/23]`, etc. after each result.

#### 3.4 Handle Batch Transitions

Between batches, optionally display:
```
Checking next batch...
```

This helps users understand why there might be brief pauses.

### Step 4: Display Final Summary

After all subagents complete (or timeout after 5 minutes), display summary section:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

12 implemented ✅
2 in review ⚠️
7 need implementation ❌
1 not applicable 🚫
1 error ⚠️

Next Steps:
→ Implement in: flutter, rust, elixir, native, unity, unreal, godot
→ Review PRs: java (#5678), android (#9012)
→ Retry: cordova (wait 60 min for rate limit reset)
```

**Summary calculation:**
- Count results by status
- List SDK names for actionable items
- Include PR numbers for "needs_review" items
- Provide specific guidance for errors (wait time, auth check, etc.)

**If timeout reached:**
```
⚠️  Timeout: 5 minutes reached, showing partial results.
Missing: {sdk_list}
Retry with: /sdk-feature-status {feature_name}
```

## Error Handling

| Failure Mode | Behavior |
|--------------|----------|
| Single SDK fails | Mark as `error`, continue others |
| Multiple SDKs fail | Report partial results, suggest retry |
| Rate limited | Return partial results, wait 60 min |
| All SDKs fail | Skill fails, check `gh auth status` |

## Verification

After generating the report:

1. **Spot-check 2-3 results**: Click PR links to verify they're actually about the feature
2. **Check "not_implemented" SDKs**: Search GitHub UI manually for 1-2 to confirm no false negatives
3. **Review "needs_review" items**: These require human judgment

| Issue Type | Common Causes |
|------------|---------------|
| False positives | PRs that mention but don't implement; similar naming, different purpose |
| False negatives | Feature under different name; implementation in unsearched sub-package |

## Search Command Reference

```bash
# PR search (all states - omit --state flag)
gh search prs "keywords" --repo {repo} --limit 10 --json number,title,state,url,mergedAt
gh search prs "sentry-docs/pull/123" org:getsentry --json url,repository,number,mergedAt
gh search prs "in:title keywords is:merged" --repo {repo} --json number,title,url,mergedAt

# PR detail fetch (for pattern extraction)
gh pr view {pr_number} --repo {repo} --json title,body,files

# Code search (without path filter)
gh search code "class ClassName" --repo {repo} --json path,repository
gh search code "config_option" --repo {repo} --json path,repository

# Code search with path filter (CRITICAL for shared repos)
gh search code "ClientReport path:sentry-android/" --repo getsentry/sentry-java --json path,repository
gh search code "ClientReport path:sentry/ -path:sentry-android/" --repo getsentry/sentry-java --json path,repository

# Issue search
gh search issues "feature name" --repo {repo} --json number,title,state,url
```

## Guidelines

**Status determination:**

| Status | Criteria |
|--------|----------|
| ✅ Implemented | Merged PR + code evidence IN THE CORRECT PATH |
| ⚠️ Needs review | Open PR OR code found but unclear |
| ❌ Not implemented | No PR, no code, no issues IN THE CORRECT PATH |
| 🚫 Not applicable | Feature technically impossible or out of scope for SDK type |
| ⚠️ Error | gh command failed (network, auth, rate limit) |

**Path filtering (CRITICAL for accuracy):**
- **Always use path filters** when specified in the SDK list
- Android and Java share `getsentry/sentry-java` but have separate codebases
- Without path filters, searches return identical results for both SDKs
- Code in `sentry-android/` does NOT mean Java SDK has the feature
- Code in `sentry/` does NOT mean Android SDK has the feature

**Performance:**

| Metric | Value |
|--------|-------|
| Parallelization | 2 batches of 8-9 subagents |
| Per SDK check | 10-30 seconds |
| Total duration | 2-5 minutes |

**Not Applicable logic:**

| Scenario | Example |
|----------|---------|
| Backend-only feature | Mobile SDKs might be N/A |
| Mobile-only feature | Backend SDKs might be N/A |
| Missing SDK APIs | Profiling features where SDK lacks profiler APIs |
| Default behavior | When unsure, prefer "not_implemented" over "not_applicable" |

## Limitations

- GitHub API rate limits: 5,000/hour (authenticated), 60/hour (unauthenticated)
- Search accuracy depends on PR titles and code patterns
- Cannot access private repos or uncommitted work
- Results are point-in-time snapshots
- Manual validation recommended for edge cases

## Dependencies

**Required:**
- GitHub CLI (`gh`) must be installed and authenticated
- Verify with: `gh auth status`

**Authentication:**
The skill requires GitHub API access to search PRs, code, and issues across Sentry SDK repositories. Authenticated access provides 5,000 requests/hour vs 60/hour unauthenticated.
