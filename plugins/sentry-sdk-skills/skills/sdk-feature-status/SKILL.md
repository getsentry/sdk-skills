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
**Output:** Progressive results as SDKs complete, then final summary (may buffer depending on execution environment)
**Time:** 2-5 minutes total

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

# Incremental retry (re-check only failed/missing SDKs)
/sdk-feature-status --retry /tmp/sdk-feature-status-1234567890.json
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

**If --retry flag with cache file:**
1. Load cached results from file: `cat /tmp/sdk-feature-status-*.json`
2. Extract feature name, patterns, and previous results from cache
3. Skip to Step 3 (re-check failed/missing SDKs only)

**If URL, extract feature name:**

| URL Type | Example | Extraction Method |
|----------|---------|-------------------|
| Docs PR | `github.com/getsentry/sentry-docs/pull/12345` | Use `gh pr view 12345 --repo getsentry/sentry-docs --json title`, extract from title |
| Develop doc | `develop.sentry.dev/sdk/profiling/` | Extract from URL path: last segment before trailing slash ("profiling") |
| Doc with anchor | `develop.sentry.dev/sdk/features/#session-replay` | Use anchor fragment ("session-replay"), ignore path |

**Priority:** If anchor present, use anchor. Otherwise use path. Page title fetching not required (would need `curl` + HTML parsing).

**If question, extract keywords:**
- Example: "Which SDKs have continuous profiling?" → keywords: `["continuous profiling"]`

**Note:** Anchors are critical for pages documenting multiple features. Using `#session-replay` targets that specific feature instead of the entire page.

### Step 2: Find Reference Implementation and Extract Patterns

This step is critical for accuracy. Extract search patterns from a reference implementation to find the feature across all SDKs.

#### 2.1 Find Reference PR

Search for best reference SDK PR:
```bash
# By develop doc (if PR link is in the doc)
gh search prs "sentry-docs/pull/12345" org:getsentry --json url,repository,number,closedAt

# By keywords (search merged PRs only)
gh search prs "client reports is:merged" org:getsentry --limit 20 --json url,repository,number,title,closedAt
```

Filter results:
- **Include**: SDK repos from the SDK List table
- **Exclude**: `getsentry/sentry`, `getsentry/sentry-docs`, `getsentry/develop`

**Selection heuristic:**
1. Sort results by `closedAt` date (latest first)
2. Select first result (latest merged PR)
   - Rationale: Later PRs have refined naming, better documentation, lessons learned

**Validation in Step 2.2:**
- After fetching PR, check if `body` field is non-empty
- If empty/null, skip pattern extraction and proceed to Step 3 with keywords only

If no PRs found at all, skip to Step 3 using only the keywords from Step 1.

#### 2.2 Fetch PR Details

Get full PR information:
```bash
gh pr view {pr_number} --repo {repo} --json title,body,files
```

This returns:
- `title`: PR title (e.g., "Add Client Reports to Python SDK")
- `body`: PR description with code examples (may be empty/null)
- `files`: List of changed files with paths

**Validation:** If `body` is empty/null, skip Step 2.3 and proceed to Step 3 using only keywords from Step 1.

#### 2.3 Extract Patterns

Extract search patterns from the reference PR to find the feature across all SDKs.

**High-level process:**
1. Extract feature name from PR title (remove prefixes like "feat:", "Add ", SDK references)
2. Extract code patterns from PR body (class names, function names, config keys)
3. Extract patterns from file paths (filenames without extensions)
4. Generate language variants (CamelCase, snake_case, kebab-case)
5. Score patterns by frequency (title: +5, body: +1, files: +3)
6. Select top patterns: 5 code_patterns, 3 config_options, all keywords

**Example output:**
```json
{
  "keywords": ["client reports"],
  "code_patterns": ["ClientReport", "ClientReportManager", "client_report"],
  "config_options": ["sendClientReports", "send_client_reports"]
}
```

**Fallback:** If <3 total patterns found, generate variants from feature name and continue.

**Detailed algorithm:** MUST follow the complete procedure in `reference/pattern-extraction.md` for extraction rules, regex patterns, filtering logic, and frequency counting.

### Step 3: Check All SDKs with Streaming Output

Launch subagents in parallel batches while displaying results incrementally as they complete.

**⚠️ Streaming Limitations**: Due to agent execution model, results may buffer and appear in batches rather than true real-time streaming. The instructions below describe the ideal output format; actual display timing depends on the execution environment.

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

**Batch generation algorithm:**

1. Get all SDKs from SDK List table (Section: SDK List)
2. Calculate batch size: `batch_size = 10`
3. Calculate number of batches: `num_batches = ceil(total_sdks / batch_size)`
4. Split SDKs into batches:
   - Batch 1: SDKs 1-10 from table
   - Batch 2: SDKs 11-20 from table
   - Batch 3: SDKs 21+ from table
   - etc.

This automatically adjusts when SDKs are added/removed from the table.

**For each SDK in each batch, launch a subagent:**

Use Task tool with:
- `subagent_type: "general-purpose"`
- `description: "Check {sdk_name} SDK for {feature_name}"`
- `prompt: "Follow the procedure in agents/sdk-checker.md with inputs: sdk_name={name}, repo={repo}, path_filter={filter}, keywords={keywords}, code_patterns={patterns}, config_options={options}"`

**Execution strategy:**
- Execute batches sequentially (Batch 1 → Batch 2 → Batch 3)
- Within each batch, execute subagents in parallel
- This balances parallelism (10 concurrent) vs API rate limits

**Subagent output schema:** See `agents/sdk-checker.md` for complete procedure and expected JSON output.

#### 3.3 Display Results as They Complete

Stream results immediately as subagents return (don't wait for batch completion).

**Display format by status:**
```
✅ python  #1234  2024-01-15  [HIGH]  getsentry/sentry-python
✅ java    #5678  2024-02-01  [MED]   getsentry/sentry-java
🔄 android  #9012  (open)  getsentry/sentry-java
❌ flutter  getsentry/sentry-dart
🚫 native  getsentry/sentry-native  (reason)
⚠️  rust  getsentry/sentry-rust  (error message)
```

**Confidence levels** (for "implemented" only):
- **HIGH**: PR + code + config | **MED**: PR + code OR code only | **LOW**: PR only or weak signal

Optionally show progress counter: `[3/23]`, `[4/23]`. Between batches, display: "Checking next batch..."

### Step 4: Display Final Summary

After all subagents complete (or timeout after 10 minutes), display summary section:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

12 implemented ✅
2 in review 🔄
7 need implementation ❌
1 not applicable 🚫
1 error ⚠️

Next Steps:
→ Implement in: flutter, rust, elixir, native, unity, unreal, godot
→ Review PRs: java (#5678), android (#9012)
→ Retry: cordova (wait 60 min for rate limit reset)
```

**Summary calculation:**
- Count by status, list actionable SDKs, include PR numbers for "needs_review"

#### 4.1 Save Results & Incremental Retry

Save results to `/tmp/sdk-feature-status-{timestamp}.json` with this schema:

```json
{
  "feature_name": "client reports",
  "patterns": {
    "keywords": ["client reports"],
    "code_patterns": ["ClientReport", "client_report"],
    "config_options": ["sendClientReports"]
  },
  "results": [
    {"name": "python", "status": "implemented", "confidence": "high", "pr_number": 1234, ...},
    {"name": "java", "status": "error", "error": "rate limited", ...}
  ]
}
```

Display cache path to user.

**On timeout/errors, suggest retry:**
```
⚠️  Timeout reached. Results cached to: /tmp/sdk-feature-status-1234567890.json
Retry: /sdk-feature-status --retry /tmp/sdk-feature-status-1234567890.json
```

**Retry logic:** Load cache → identify failed/missing SDKs → re-check only those → merge → display updated summary. Saves 2-5 minutes.

## Error Handling

| Failure Mode | Behavior |
|--------------|----------|
| Single SDK fails | Mark as `error`, continue others |
| Multiple SDKs fail | Report partial results, suggest retry |
| Rate limited | Return partial results, wait 60 min |
| All SDKs fail | Skill fails, check `gh auth status` |

## Search Command Reference

```bash
# PR search (all states - omit --state flag)
gh search prs "keywords" --repo {repo} --limit 10 --json number,title,state,url,closedAt
gh search prs "sentry-docs/pull/123" org:getsentry --json url,repository,number,closedAt
gh search prs "in:title keywords is:merged" --repo {repo} --json number,title,url,closedAt

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

**Status determination:** ✅ Implemented (PR + code in correct path, confidence: HIGH/MED/LOW) | 🔄 Needs review (open PR or unclear) | ❌ Not implemented (no evidence) | 🚫 Not applicable (impossible for SDK type) | ⚠️ Error (command failed)

**Confidence:** HIGH (PR + code + config) | MED (PR + code OR code only) | LOW (PR only or weak signal)

**Path filtering:** CRITICAL for Java/Android sharing `getsentry/sentry-java`. Always use path filters from SDK table—code in `sentry-android/` ≠ Java has feature.

**Performance:** ceil(sdks/10) batches × 10 parallel, 10-30s per SDK, 2-5min total

**Not applicable:** Use for backend-only/mobile-only features or missing SDK APIs. Default: prefer "not_implemented" over "not_applicable" when unsure.

## Limitations

- GitHub API rate limits: 5,000/hour (authenticated), 60/hour (unauthenticated)
- Search accuracy depends on PR titles and code patterns
- Cannot access private repos or uncommitted work
- Results are point-in-time snapshots

## Dependencies

**Required:**
- GitHub CLI (`gh`) must be installed and authenticated
- Verify with: `gh auth status`

**Authentication:**
The skill requires GitHub API access to search PRs, code, and issues across Sentry SDK repositories. Authenticated access provides 5,000 requests/hour vs 60/hour unauthenticated.
