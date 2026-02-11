---
name: sdk-feature-status
description: Use this skill when the user asks about SDK feature implementation status, SDK parity, or cross-SDK coverage. Triggers include: develop doc URLs (github.com/getsentry/sentry-docs/...), questions like 'which SDKs have X?', 'is X implemented in Python SDK?', 'SDK parity for feature Y', or requests to check feature rollout status. Do NOT use for: SDK usage questions (how to use a feature), bug reports, SDK configuration help, or questions about a single SDK's behavior.
allowed-tools: Bash Task
---

# SDK Feature Status Checker

Check which of the 17 Sentry SDKs have implemented a feature.

## Usage

**Input:** Develop doc URL or free-form question
**Output:** Formatted table + optional machine-readable JSON
**Time:** 2-5 minutes

**Examples:**
```bash
/sdk-feature-status https://github.com/getsentry/sentry-docs/pull/12345
/sdk-feature-status Which SDKs have continuous profiling?
```

## Output Schema

### feature-status.json (optional)
```json
{
  "feature": "client-reports",
  "timestamp": "2024-01-15T12:00:00Z",
  "develop_doc": {"url": "https://...", "type": "pr"},
  "reference_implementation": {
    "sdk": "javascript",
    "repo": "getsentry/sentry-javascript",
    "pr_url": "https://...",
    "pr_number": 6789
  },
  "search_patterns": {
    "keywords": ["client reports"],
    "code_patterns": ["class ClientReportManager"],
    "config_options": ["sendClientReports"]
  },
  "summary": {
    "implemented": 5,
    "needs_review": 2,
    "not_implemented": 8,
    "not_applicable": 1,
    "errors": 1
  },
  "sdks": [
    {
      "name": "python",
      "repo": "getsentry/sentry-python",
      "path_filter": null,
      "status": "implemented|needs_review|not_implemented|not_applicable|error",
      "pr_url": "https://..." or null,
      "pr_number": 1234 or null,
      "merged_date": "2024-01-15" or null,
      "notes": "Brief summary",
      "error": null or "error message"
    },
    {
      "name": "android",
      "repo": "getsentry/sentry-java",
      "path_filter": "path:sentry-android/",
      "status": "implemented",
      "pr_url": "https://...",
      "pr_number": 1234,
      "merged_date": "2024-01-15",
      "notes": "Found in sentry-android/ directory",
      "error": null
    }
  ]
}
```

## SDK List (Check All 17)

| SDK | Repository | Type | Path Filter |
|-----|------------|------|-------------|
| `javascript` | `getsentry/sentry-javascript` | Frontend | - |
| `python` | `getsentry/sentry-python` | Backend | - |
| `java` | `getsentry/sentry-java` | Backend | `path:sentry/` |
| `ruby` | `getsentry/sentry-ruby` | Backend | - |
| `php` | `getsentry/sentry-php` | Backend | - |
| `go` | `getsentry/sentry-go` | Backend | - |
| `dotnet` | `getsentry/sentry-dotnet` | Backend | - |
| `rust` | `getsentry/sentry-rust` | Backend | - |
| `elixir` | `getsentry/sentry-elixir` | Backend | - |
| `cocoa` | `getsentry/sentry-cocoa` | Mobile | - |
| `android` | `getsentry/sentry-java` | Mobile | `path:sentry-android/` |
| `react-native` | `getsentry/sentry-react-native` | Mobile | - |
| `flutter` | `getsentry/sentry-dart` | Mobile | - |
| `kotlin` | `getsentry/sentry-kotlin-multiplatform` | Mobile | - |
| `unity` | `getsentry/sentry-unity` | Gaming | - |
| `unreal` | `getsentry/sentry-unreal` | Gaming | - |
| `native` | `getsentry/sentry-native` | Native | - |

**Note:** Path filters are required for SDKs that share repositories. Android and Java both use `getsentry/sentry-java` but have separate codebases in different directories.

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

Launch 17 subagents (batches of 8-10) with this prompt template:

```
Check if {sdk_name} has implemented {feature_name}.

Repository: {repo}
Path filter: {path_filter} (empty if not needed)
Search patterns:
- Keywords: {keywords}
- Code: {code_patterns}
- Config: {config_options}

Execute these searches:

1. PR search (filter by path in PR title/body if path_filter provided):
gh search prs --repo {repo} "{keywords}" --state all --limit 10 --json number,title,state,url,mergedAt

2. Code search (CRITICAL: include path filter to avoid false matches):
   - If path_filter is provided: gh search code --repo {repo} "{code_pattern} {path_filter}" --json path,repository
   - If no path_filter: gh search code --repo {repo} "{code_pattern}" --json path,repository

   Example for android: gh search code --repo getsentry/sentry-java "ClientReport path:sentry-android/" --json path,repository
   Example for java: gh search code --repo getsentry/sentry-java "ClientReport path:sentry/" --json path,repository

3. Issue search (if nothing found):
gh search issues --repo {repo} "{keywords}" --json number,title,state,url

IMPORTANT: When path_filter is provided, ONLY count code matches within that path.
- For android (path:sentry-android/): ONLY files in sentry-android/ directory
- For java (path:sentry/): ONLY files in sentry/ directory (NOT sentry-android/)

Determine status:
- ✅ implemented: Merged PR + code evidence found IN THE CORRECT PATH
- ⚠️ needs_review: Open PR or unclear implementation
- ❌ not_implemented: No evidence found IN THE CORRECT PATH
- 🚫 not_applicable: Feature doesn't apply to this SDK type
- ⚠️ error: gh command failed (include error message)

Return this exact JSON:
{
  "sdk": "{sdk_name}",
  "repo": "{repo}",
  "path_filter": "{path_filter}" or null,
  "status": "implemented|needs_review|not_implemented|not_applicable|error",
  "pr_url": "https://..." or null,
  "pr_number": 1234 or null,
  "merged_date": "2024-01-15" or null,
  "notes": "Brief finding (1 sentence, mention path if filtered)",
  "error": null or "error message"
}
```

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

### Step 5: Optional Machine-Readable Output

After displaying the formatted report, ask the user:

**"Would you like me to save a machine-readable JSON report?"**

If yes, create `.sdk-align/` directory in current working directory and write `feature-status.json`:

```json
{
  "feature": "<name>",
  "timestamp": "<ISO8601>",
  "develop_doc": {"url": "<url>", "type": "pr|commit|url"},
  "reference_implementation": {
    "sdk": "<sdk>",
    "repo": "<repo>",
    "pr_url": "<url>",
    "pr_number": <num>
  },
  "search_patterns": {
    "keywords": ["<kw1>", "<kw2>"],
    "code_patterns": ["<pattern1>"],
    "config_options": ["<opt1>"]
  },
  "summary": {
    "implemented": <count>,
    "needs_review": <count>,
    "not_implemented": <count>,
    "not_applicable": <count>,
    "errors": <count>
  },
  "sdks": [<array of subagent results>]
}
```

Then confirm:
```
✅ Saved: .sdk-align/feature-status.json
```

## Error Handling

**SDK check fails:**
- Mark status as `"error"` with error message
- Continue checking other SDKs
- Include in final report

**Multiple failures:**
- Report succeeds if ≥1 SDK succeeds
- Show errors in report
- Suggest retry

**Rate limiting:**
- Return partial results
- Mark remaining as "error: Rate limit exceeded"
- Suggest: wait 60 min or use authenticated gh CLI

**All checks fail:**
- Skill fails with clear error
- Check: `gh auth status`, network, API limits

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
gh search code "ClientReport path:sentry/" --repo getsentry/sentry-java --json path,repository

# Issue search
gh search issues "feature name" --repo {repo} --json number,title,state,url
```

## Guidelines

**Status determination:**
- **Implemented:** Merged PR + code evidence IN THE CORRECT PATH
- **Needs review:** Open PR OR code found but unclear
- **Not implemented:** No PR, no code, no issues IN THE CORRECT PATH
- **Not applicable:** Feature technically impossible or out of scope for SDK type
- **Error:** gh command failed (network, auth, rate limit)

**Path filtering (CRITICAL for accuracy):**
- **Always use path filters** when specified in the SDK list
- Android and Java share `getsentry/sentry-java` but have separate codebases
- Without path filters, searches return identical results for both SDKs
- Code in `sentry-android/` does NOT mean Java SDK has the feature
- Code in `sentry/` does NOT mean Android SDK has the feature

**Performance:**
- Launch subagents in parallel (2 batches of 8-9 each)
- Use `--limit` flags to avoid excessive API calls
- Each SDK check: 10-30 seconds
- Total: 2-5 minutes

**Not Applicable logic:**
- Backend-only feature → mobile SDKs might be N/A
- Mobile-only feature → backend SDKs might be N/A
- Profiling features → some SDKs lack profiler APIs
- When unsure, prefer "not_implemented" over "not_applicable"

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
