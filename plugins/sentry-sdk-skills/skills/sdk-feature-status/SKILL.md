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
| `java` | `getsentry/sentry-java` | Backend | `path:sentry/ -path:sentry-android` | Shares repo with Android |
| `ruby` | `getsentry/sentry-ruby` | Backend | - | Full feature support |
| `php` | `getsentry/sentry-php` | Backend | - | Full feature support |
| `go` | `getsentry/sentry-go` | Backend | - | Full feature support |
| `dotnet` | `getsentry/sentry-dotnet` | Backend | - | Full feature support |
| `rust` | `getsentry/sentry-rust` | Backend | - | Full feature support |
| `elixir` | `getsentry/sentry-elixir` | Backend | - | Community-maintained |
| `laravel` | `getsentry/sentry-laravel` | Framework | - | PHP framework integration |
| `symfony` | `getsentry/sentry-symfony` | Framework | - | PHP framework integration |
| `cocoa` | `getsentry/sentry-cocoa` | Mobile | - | iOS/macOS, profiling supported |
| `android` | `getsentry/sentry-java` | Mobile | `path:sentry-android-core` | Shares repo with Java, profiling supported |
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
1. Extract the file path from the user's --retry argument (e.g., `/tmp/sdk-feature-status-1234567890.json`)
2. Load cached results from that specific file: `cat /tmp/sdk-feature-status-1234567890.json`
3. Parse JSON to extract feature name, patterns, and previous results from cache
4. Skip to Step 3 (re-check only SDKs with status "error" or missing from cache)

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

Search for best reference SDK PR (merged PRs only):
```bash
# By develop doc (if PR link is in the doc)
gh search prs "sentry-docs/pull/12345 is:merged" org:getsentry --json url,repository,number,closedAt

# By keywords (search merged PRs only)
gh search prs "client reports is:merged" org:getsentry --limit 20 --json url,repository,number,title,closedAt
```

**Note:** The `is:merged` qualifier ensures we only get merged PRs as reference implementations, excluding rejected/closed PRs.

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
- `title`: PR title (e.g., "Add Client Reports to Python SDK") - always available
- `body`: PR description with code examples (may be empty/null)
- `files`: List of changed files with paths - always available

**Note:** Even if `body` is empty/null, proceed to Step 2.3 to extract patterns from `title` and `files`.

#### 2.3 Extract Patterns

Extract search patterns from the reference PR to find the feature across all SDKs.

**High-level process:**
1. Extract feature name from PR title (remove prefixes like "feat:", "Add ", SDK references)
2. Extract code patterns from PR body **if body is non-empty** (class names, function names, config keys)
3. Extract patterns from file paths (filenames without extensions)
4. Generate language variants (CamelCase, snake_case, kebab-case)
5. Score patterns by frequency (title: +5, body: +1 if available, files: +3)
6. Select top patterns: 5 code_patterns, 3 config_options, all keywords

**Handling empty PR body:**
- If `body` is empty/null, skip body pattern extraction (step 2 above)
- Still extract patterns from `title` (step 1) and `files` (step 3)
- Continue with title-derived keywords and file-path patterns
- This preserves valuable signals even when PR description is missing

**Example output:**
```json
{
  "keywords": ["client reports"],
  "code_patterns": ["ClientReport", "ClientReportManager", "client_report"],
  "config_options": ["sendClientReports", "send_client_reports"]
}
```

**Fallback:** If <3 total patterns found (after extracting from title and files), generate variants from feature name and continue.

**Empty body is not a failure:** Title + file patterns often provide sufficient signal (e.g., PR title "Add Client Reports" + files like "ClientReport.java" yields strong patterns).

**Detailed algorithm:** MUST follow the complete procedure in `reference/pattern-extraction.md` for extraction rules, regex patterns, filtering logic, and frequency counting.

### Step 3: Check All SDKs with Streaming Output

Launch subagents in parallel batches while displaying results incrementally as they complete.

**⚠️ Streaming Limitations**: Due to agent execution model, results may buffer and appear in batches rather than true real-time streaming. The instructions below describe the ideal output format; actual display timing depends on the execution environment.

#### 3.1 Display Initial Header

Show header immediately before launching subagents using GitHub-flavored markdown:

```markdown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SDK Feature Status: `{feature_name}`
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Reference**: `{sdk}` ([`{repo}`#{pr}](https://github.com/{repo}/pull/{pr})), merged `{date}`
**Develop Doc**: [{url}]({url})

Checking **{total}** SDKs...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Example:**
```markdown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SDK Feature Status: `client reports`
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Reference**: `python` ([`getsentry/sentry-python`#1234](https://github.com/getsentry/sentry-python/pull/1234)), merged `2024-01-15`
**Develop Doc**: [https://develop.sentry.dev/sdk/features/](https://develop.sentry.dev/sdk/features/)

Checking **23** SDKs...

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

Each result line follows this format (use GitHub-flavored markdown):
- ✅ **implemented**: `{sdk}` [#{pr_number}]({pr_url}) `{merged_date}` `[{CONFIDENCE}]` `{repo}`
- 🔄 **needs_review**: `{sdk}` [#{pr_number}]({pr_url}) *(open)* `{repo}`
- ❌ **not_implemented**: `{sdk}` `{repo}`
- 🚫 **not_applicable**: `{sdk}` `{repo}` *({reason})*
- ⚠️ **error**: `{sdk}` `{repo}` *({error_message})*

**Example output:**
```markdown
✅ `python` [#1234](https://github.com/getsentry/sentry-python/pull/1234) `2024-01-15` `[HIGH]` `getsentry/sentry-python`
✅ `java` [#5678](https://github.com/getsentry/sentry-java/pull/5678) `2024-02-01` `[MED]` `getsentry/sentry-java`
🔄 `android` [#9012](https://github.com/getsentry/sentry-java/pull/9012) *(open)* `getsentry/sentry-java`
❌ `flutter` `getsentry/sentry-dart`
🚫 `native` `getsentry/sentry-native` *(C/C++ SDK, no HTTP instrumentation)*
⚠️ `rust` `getsentry/sentry-rust` *(rate limited)*
```

**Formatting notes:**
- SDK names, dates, confidence levels, and repos use inline code (backticks)
- PR links: Use agent's `pr_url` if available, otherwise construct from `repo` and `pr_number`: `https://github.com/{repo}/pull/{pr_number}`
- Reasons/errors use italics for secondary information
- All markdown must render correctly on GitHub

**Confidence levels** (for "implemented" only):
- `[HIGH]`: PR + code + config
- `[MED]`: PR + code OR code only
- `[LOW]`: PR only or weak signal

Optionally show progress counter: `[3/23]`, `[4/23]`. Between batches, display: "Checking next batch..."

### Step 4: Display Final Summary

After all subagents complete (or timeout after 10 minutes), display summary section using GitHub-flavored markdown:

```markdown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**12** implemented ✅
**2** in review 🔄
**7** need implementation ❌
**1** not applicable 🚫
**1** error ⚠️

**Next Steps:**
- **Implement in**: `flutter`, `rust`, `elixir`, `native`, `unity`, `unreal`, `godot`
- **Review PRs**: `java` ([#5678](https://github.com/getsentry/sentry-java/pull/5678)), `android` ([#9012](https://github.com/getsentry/sentry-java/pull/9012))
- **Retry failed**: `cordova` *(wait 60 min for rate limit reset)*
```

**Summary calculation:**
- Count by status with bold numbers
- List actionable SDKs with inline code formatting
- Include PR links (not just numbers) for "needs_review"
- Format errors/reasons in italics

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

Display cache path to user using inline code formatting.

**On timeout/errors, suggest retry:**
```markdown
⚠️ **Timeout reached.** Results cached to: `/tmp/sdk-feature-status-1234567890.json`

**Retry command:**
```bash
/sdk-feature-status --retry /tmp/sdk-feature-status-1234567890.json
```
```

**Retry logic:**
1. Parse the user-provided cache file path from --retry argument
2. Load JSON from that specific file (do NOT use wildcards)
3. Identify SDKs to re-check: any with status="error" or completely missing from cache
4. Re-run only those SDKs (skip SDKs with status="implemented", "needs_review", "not_implemented", "not_applicable")
5. Merge new results with cached results (new results overwrite cached entries for re-checked SDKs)
6. Display updated summary with all results
7. Save merged results back to a new timestamped cache file

This saves 2-5 minutes by avoiding re-checks of successful SDKs.

## Error Handling

| Failure Mode | Behavior |
|--------------|----------|
| Single SDK fails | Mark as `error`, continue others |
| Multiple SDKs fail | Report partial results, suggest retry |
| Rate limited | Return partial results, wait 60 min |
| All SDKs fail | Skill fails, check `gh auth status` |

## Search Command Reference

```bash
# PR search (use search qualifiers to filter by state)
# IMPORTANT: Use is:merged to get only merged PRs, is:open for open PRs
# The mergedAt field is NOT available in gh search prs output
gh search prs "keywords is:merged" --repo {repo} --limit 10 --json number,title,state,url,closedAt
gh search prs "keywords is:open" --repo {repo} --limit 5 --json number,title,state,url
gh search prs "sentry-docs/pull/123 is:merged" org:getsentry --json url,repository,number,closedAt
gh search prs "in:title keywords is:merged" --repo {repo} --json number,title,url,closedAt

# PR detail fetch (for pattern extraction)
gh pr view {pr_number} --repo {repo} --json title,body,files

# Code search (without path filter)
gh search code "class ClassName" --repo {repo} --json path,repository
gh search code "config_option" --repo {repo} --json path,repository

# Code search with path filter (CRITICAL for shared repos)
gh search code "ClientReport path:sentry-android-core" --repo getsentry/sentry-java --json path,repository
gh search code "ClientReport path:sentry/ -path:sentry-android" --repo getsentry/sentry-java --json path,repository

# Issue search
gh search issues "feature name" --repo {repo} --json number,title,state,url
```

## Guidelines

**Status determination:** ✅ Implemented (evidence found: merged PR OR code OR config, confidence: HIGH/MED/LOW) | 🔄 Needs review (open PR or unclear) | ❌ Not implemented (no evidence) | 🚫 Not applicable (impossible for SDK type) | ⚠️ Error (command failed)

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
