# SDK Checker Agent

Check if a single SDK has implemented a feature.

## Input

- `sdk_name`: SDK identifier (e.g., "python", "android")
- `repo`: GitHub repository (e.g., "getsentry/sentry-python")
- `path_filter`: Path scope for shared repos (e.g., "path:sentry-android/") or null
- `keywords`: Search terms (e.g., ["client reports"])
- `code_patterns`: Code patterns to search for (e.g., ["ClientReportManager"])
- `config_options`: Config option names (e.g., ["sendClientReports"])

## Procedure

### 1. PR Search

Search all PR states (open, closed, merged) by omitting --state flag:

```bash
gh search prs --repo {repo} "{keywords}" --limit 10 --json number,title,state,url,closedAt,mergedAt
```

**CRITICAL - PR state distinction:**
- The `state` field only returns "OPEN" or "CLOSED" (merged PRs appear as "CLOSED", not "MERGED")
- Use `mergedAt` field to distinguish merged from rejected PRs:
  - **OPEN**: `state == "OPEN"` - Implementation in progress
  - **MERGED**: `state == "CLOSED" AND mergedAt != null` - Implementation complete
  - **CLOSED** (rejected): `state == "CLOSED" AND mergedAt == null` - Abandoned/rejected, ignore
- This is a known gh CLI limitation (cli/cli#6290)

**Result categorization:**
After fetching PRs, categorize into three groups:
1. Open PRs: `state == "OPEN"`
2. Merged PRs: `state == "CLOSED" AND mergedAt != null`
3. Closed (rejected) PRs: `state == "CLOSED" AND mergedAt == null` (ignore these)

Store both open and merged PRs for status determination (don't prioritize yet).

### 2. Code Search

Search for BOTH code patterns AND config options. Config options are often the most reliable indicators of feature implementation.

**Search order:** Search all code_patterns first, then all config_options. Aggregate results.

**CRITICAL**: Include path filter to avoid false matches in shared repositories.

**For each pattern in code_patterns:**
- **If path_filter is provided**:
  ```bash
  gh search code --repo {repo} "{code_pattern} {path_filter}" --json path,repository
  ```

- **If no path_filter**:
  ```bash
  gh search code --repo {repo} "{code_pattern}" --json path,repository
  ```

**For each option in config_options:**
- **If path_filter is provided**:
  ```bash
  gh search code --repo {repo} "{config_option} {path_filter}" --json path,repository
  ```

- **If no path_filter**:
  ```bash
  gh search code --repo {repo} "{config_option}" --json path,repository
  ```

**Examples for shared repo:**
```bash
# Android in getsentry/sentry-java - code pattern
gh search code --repo getsentry/sentry-java "ClientReport path:sentry-android-core" --json path,repository

# Android - config option
gh search code --repo getsentry/sentry-java "sendClientReports path:sentry-android-core" --json path,repository

# Java in getsentry/sentry-java (with negative filter to exclude Android)
gh search code --repo getsentry/sentry-java "ClientReport path:sentry/ -path:sentry-android" --json path,repository
gh search code --repo getsentry/sentry-java "send_client_reports path:sentry/ -path:sentry-android" --json path,repository
```

**IMPORTANT**: When path_filter is provided, ONLY count code matches within that path.
- For android (`path:sentry-android-core`): ONLY files in `sentry-android-core/` directory (main Android SDK module)
- For java (`path:sentry/ -path:sentry-android`): ONLY files in `sentry/` directory, excluding ALL `sentry-android*` directories
- The negative filter `-path:sentry-android` (without trailing slash) is CRITICAL - it excludes all Android directories (sentry-android-core, sentry-android-ndk, etc.) as a prefix match

**Result aggregation:**
- Track whether code patterns were found: `has_code_pattern = true/false`
- Track whether config options were found: `has_config_option = true/false`
- Use both for confidence calculation (see Output section)

### 3. Issue Search

If nothing found in PR or code search:

```bash
gh search issues --repo {repo} "{keywords}" --json number,title,state,url
```

## Output

Return exactly this JSON structure:

```json
{
  "name": "{sdk_name}",
  "repo": "{repo}",
  "path_filter": "{path_filter}" or null,
  "status": "implemented|needs_review|not_implemented|not_applicable|error",
  "confidence": "high|medium|low" or null,
  "pr_url": "https://github.com/{repo}/pull/{pr_number}" or null,
  "pr_number": 1234 or null,
  "merged_date": "2024-01-15" or null,
  "notes": "Brief finding (1 sentence, mention path if filtered)",
  "error": null or "error message"
}
```

**Field requirements:**
- `pr_url`: MUST be the full GitHub URL when PR found (use `url` field from `gh search prs` output)
- `pr_number`: Extract from PR URL or use `number` field from search results
- `merged_date`: Use `closedAt` field from search results, format as YYYY-MM-DD
- `notes`: Keep concise, use for reasons (not_applicable) or context (implemented with path filter)

**Confidence field** (only for status = "implemented"):
- **high**: Merged PR found + code patterns found + config options found
- **medium**: Merged PR + code patterns found (no config options), OR code patterns/config options found but no PR, OR config options found alone (config options are highly distinctive)
- **low**: Merged PR mention only (no code evidence), OR single weak signal, OR only generic code patterns found
- **null**: For all other statuses (needs_review, not_implemented, not_applicable, error)

**Note**: Config options alone can justify "medium" confidence because option names like `sendClientReports` or `maxRequestBodySize` are highly distinctive and unlikely to appear unless the feature is implemented.

## Status Determination

Apply in this order:

1. **⚠️ error**: If ANY gh command failed (rate limit, auth issue, network error)
   - confidence = null, include error message in notes

2. **✅ implemented**: If ANY strong evidence found IN THE CORRECT PATH:
   - Merged PR (state == "CLOSED" AND mergedAt != null), OR
   - Code pattern matches, OR
   - Config option matches
   - Set confidence based on evidence strength (see Output section)
   - **This takes precedence over open PRs**: If both merged PR and open PR exist, classify as implemented (open PR is likely follow-up work)
   - When only weak evidence exists, prefer "implemented" with low confidence over "not_implemented"
   - Note: CLOSED (rejected) PRs (mergedAt == null) don't count as evidence

3. **🔄 needs_review**: If open PR found (state == "OPEN") AND no implementation evidence
   - Implementation in progress but not merged
   - Only use if Step 2 found NO merged PR, NO code patterns, NO config options
   - If merged PR or code exists alongside open PR, use "implemented" instead (open PR is likely unrelated or follow-up)
   - confidence = null

4. **🚫 not_applicable**: If feature doesn't apply to this SDK type
   - Examples: backend-only feature on mobile SDK, server-side feature on client SDK
   - confidence = null
   - Only use when certain the feature is impossible for this SDK type

5. **❌ not_implemented**: If no evidence found IN THE CORRECT PATH
   - No open/merged PRs, no code patterns, no config options
   - Closed (rejected) PRs only (mergedAt == null)
   - confidence = null
   - Default when searches return empty results

### Example Decision Flows

**Scenario A: MERGED PR + code found**
- Merged PR (mergedAt != null) + code patterns found
- Result: ✅ implemented (high confidence)

**Scenario B: OPEN PR + no other evidence**
- Open PR found, no merged PR, no code/config
- Result: 🔄 needs_review

**Scenario C: MERGED PR + OPEN PR**
- Merged PR (mergedAt != null) + open PR found
- Result: ✅ implemented (merged evidence takes precedence, open PR likely follow-up)
- Note: This prevents false "needs_review" from unrelated open PRs

**Scenario D: Code found + OPEN PR**
- Code patterns/config found + open PR
- Result: ✅ implemented (code evidence takes precedence)
- Note: Open PR might be tangential or adding more to existing implementation

**Scenario E: CLOSED (rejected) PR + code found**
- Closed PR (mergedAt == null), but code patterns found
- Result: ✅ implemented (medium confidence - code only)
- Note: PR was rejected but feature was implemented differently

**Scenario F: CLOSED (rejected) PR + no code**
- Closed PR (mergedAt == null), no code/config, no other PRs
- Result: ❌ not_implemented
- Note: Rejected PR means abandoned attempt, feature not implemented

**Scenario G: MERGED PR only (no code)**
- Merged PR (mergedAt != null), no code patterns found
- Result: ✅ implemented (low confidence - PR only)
- Note: Code search may have missed patterns, or code was in different format

## Path Filtering Rules

**Critical for accuracy in shared repositories:**

- Always use path filters when provided
- Android and Java share `getsentry/sentry-java` but have separate codebases
- Without path filters, searches return identical results for both SDKs
- Code in `sentry-android-core/` does NOT mean Java SDK has the feature
- Code in `sentry/` does NOT mean Android SDK has the feature

**Android SDK path structure:**
- Main Android code is in `sentry-android-core/` (NOT `sentry-android/`)
- Additional modules: `sentry-android-ndk/`, `sentry-android-replay/`, etc.
- The `sentry-android/` directory itself contains no SDK code (just build config)
- Use `path:sentry-android-core` for Android searches

**Prefix matching for exclusion:**
- GitHub's negative path filter uses prefix matching
- `-path:sentry-android` (no trailing slash) excludes ALL directories starting with `sentry-android`
- This excludes: `sentry-android/`, `sentry-android-core/`, `sentry-android-ndk/`, etc.
- Example: `path:sentry/ -path:sentry-android` ensures Java-only results (excludes all Android)
