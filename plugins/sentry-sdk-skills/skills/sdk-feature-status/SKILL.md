---
name: sdk-feature-status
description: Use this skill when the user asks about SDK feature implementation status, SDK parity, or cross-SDK coverage. Triggers include: develop doc URLs (github.com/getsentry/sentry-docs/...), questions like 'which SDKs have X?', 'does Python SDK have X?', 'SDK parity for feature Y', or requests to check feature rollout status. Note: Even single-SDK queries will check all 17 SDKs to provide parity context. Do NOT use for: SDK usage questions (how to use a feature), bug reports, SDK configuration help, or runtime behavior questions.
allowed-tools: Bash Task
---

# SDK Feature Status Checker

Check which of the 17 Sentry SDKs have implemented a feature.

## Quick Reference

| Input | Action |
|-------|--------|
| Develop doc URL | Extract feature, find reference PR, check all SDKs |
| Feature question | Extract keywords, search for reference implementation, check all SDKs |

| Output | Location |
|--------|----------|
| Formatted report | Displayed inline |

**Coverage:** 17 SDKs (JavaScript, Python, Java, Android, Ruby, PHP, Go, .NET, Rust, Elixir, Cocoa, React Native, Flutter, Kotlin, Unity, Unreal, Native)
**Duration:** 2-5 minutes
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
/sdk-feature-status https://github.com/getsentry/sentry-docs/pull/12345
/sdk-feature-status Which SDKs have continuous profiling?
```

## SDK List (Check All 17)

| SDK | Repository | Type | Path Filter | Notes |
|-----|------------|------|-------------|-------|
| `javascript` | `getsentry/sentry-javascript` | Frontend | - | Browser + Node.js environments |
| `python` | `getsentry/sentry-python` | Backend | - | Full feature support |
| `java` | `getsentry/sentry-java` | Backend | `path:sentry/ -path:sentry-android/` | Shares repo with Android |
| `ruby` | `getsentry/sentry-ruby` | Backend | - | Full feature support |
| `php` | `getsentry/sentry-php` | Backend | - | Full feature support |
| `go` | `getsentry/sentry-go` | Backend | - | Full feature support |
| `dotnet` | `getsentry/sentry-dotnet` | Backend | - | Full feature support |
| `rust` | `getsentry/sentry-rust` | Backend | - | Full feature support |
| `elixir` | `getsentry/sentry-elixir` | Backend | - | Community-maintained |
| `cocoa` | `getsentry/sentry-cocoa` | Mobile | - | iOS/macOS, profiling supported |
| `android` | `getsentry/sentry-java` | Mobile | `path:sentry-android/` | Shares repo with Java, profiling supported |
| `react-native` | `getsentry/sentry-react-native` | Mobile | - | Wraps JavaScript + native SDKs |
| `flutter` | `getsentry/sentry-dart` | Mobile | - | Cross-platform mobile |
| `kotlin` | `getsentry/sentry-kotlin-multiplatform` | Mobile | - | Multiplatform support |
| `unity` | `getsentry/sentry-unity` | Gaming | - | Game engine specific |
| `unreal` | `getsentry/sentry-unreal` | Gaming | - | Game engine specific, C++ based |
| `native` | `getsentry/sentry-native` | Native | - | C/C++, limited feature set |

**Note:** Path filters are required for SDKs sharing repositories. Notes column provides capability hints for "Not Applicable" status decisions.

## Implementation

### Step 1: Parse Input

- **If URL:** Extract feature from PR/doc title
- **If question:** Extract keywords (e.g., "continuous profiling")

### Step 2: Find Reference Implementation

Search for earliest merged PR:
```bash
# By develop doc
gh search prs "sentry-docs/pull/12345" org:getsentry --json url,repository,number,mergedAt

# By keywords
gh search prs "client reports" org:getsentry --state merged --limit 20 --json url,repository,number,title,mergedAt
```

Filter to SDK repos only (exclude `sentry`, `sentry-docs`, `develop`).

Extract patterns from reference PR:
- Class names: `ClientReportManager`, `ClientReportRecorder`
- Config options: `sendClientReports`, `send_client_reports`
- Function names: `sendClientReport`, `recordClientReport`

### Step 3: Check All SDKs in Parallel

Launch 17 subagents (batches of 8-10). See `agents/sdk-checker.md` for the subagent procedure.

**Input for each subagent:**
- SDK name, repository, and path_filter (from SDK list)
- Search patterns extracted from reference implementation (keywords, code patterns, config options)

**Output from each subagent:**
- JSON object with status, PR info, and notes
- See `agents/sdk-checker.md` for complete schema

Collect all subagent results (JSON array).

### Step 4: Generate Report

**Display formatted report:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SDK Feature Status: {feature_name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reference: {sdk} ({repo}#{pr}, merged {date})
Develop Doc: {url}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ Implemented (X)
  {sdk}  #{pr}  {date}  {repo}
  ...

⚠️  Needs Review (X)
  {sdk}  #{pr}  (open)  {repo}
  ...

❌ Not Implemented (X)
  {sdk}  {repo}
  ...

🚫 Not Applicable (X)
  {sdk}  {repo}  {reason}

⚠️  Errors (X)
  {sdk}  {repo}  {error}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

X implemented ✅
X in review ⚠️
X need implementation ❌

Next Steps:
→ Implement in: {list}
→ Review PRs: {list}
→ Retry: {list} (if errors)
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
# PR search
gh search prs "keywords" --repo {repo} --state all --limit 10 --json number,title,state,url,mergedAt
gh search prs "sentry-docs/pull/123" org:getsentry --json url,repository,number,mergedAt
gh search prs "in:title keywords" --repo {repo} --state merged

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
