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
gh search prs --repo {repo} "{keywords}" --limit 10 --json number,title,state,url,closedAt
```

**Important - PR state handling:**
- **OPEN**: Implementation in progress, use needs_review status
- **MERGED**: Implementation complete, use implemented status (with code evidence)
- **CLOSED** (not merged): Rejected/abandoned, ignore these - treat as if no PR exists

When multiple PRs found, prioritize by state: OPEN > MERGED > CLOSED

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
gh search code --repo getsentry/sentry-java "ClientReport path:sentry-android/" --json path,repository

# Android - config option
gh search code --repo getsentry/sentry-java "sendClientReports path:sentry-android/" --json path,repository

# Java in getsentry/sentry-java (with negative filter to exclude Android)
gh search code --repo getsentry/sentry-java "ClientReport path:sentry/ -path:sentry-android/" --json path,repository
gh search code --repo getsentry/sentry-java "send_client_reports path:sentry/ -path:sentry-android/" --json path,repository
```

**IMPORTANT**: When path_filter is provided, ONLY count code matches within that path.
- For android (`path:sentry-android/`): ONLY files in `sentry-android/` directory
- For java (`path:sentry/ -path:sentry-android/`): ONLY files in `sentry/` directory, explicitly excluding `sentry-android/`
- The negative filter `-path:sentry-android/` is CRITICAL to prevent false positives from substring matching

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
  "pr_url": "https://..." or null,
  "pr_number": 1234 or null,
  "merged_date": "2024-01-15" or null,
  "notes": "Brief finding (1 sentence, mention path if filtered)",
  "error": null or "error message"
}
```

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

2. **🔄 needs_review**: If open PR found (state == "OPEN")
   - Implementation in progress but not merged
   - Ignore CLOSED PRs (rejected/abandoned, not awaiting review)
   - confidence = null

3. **✅ implemented**: If ANY evidence found IN THE CORRECT PATH:
   - Merged PR (state == "MERGED"), OR
   - Code pattern matches, OR
   - Config option matches
   - Set confidence based on evidence strength (see Output section)
   - When only weak evidence exists, prefer "implemented" with low confidence over "not_implemented"
   - Note: CLOSED (non-merged) PRs don't count as evidence - check for code/config instead

4. **🚫 not_applicable**: If feature doesn't apply to this SDK type
   - Examples: backend-only feature on mobile SDK, server-side feature on client SDK
   - confidence = null
   - Only use when certain the feature is impossible for this SDK type

5. **❌ not_implemented**: If no evidence found IN THE CORRECT PATH
   - No PRs (open or merged), no code patterns, no config options
   - confidence = null
   - Default when searches return empty results

### Example Decision Flows

**Scenario A: MERGED PR + code found**
- Result: ✅ implemented (high confidence)

**Scenario B: OPEN PR found**
- Result: 🔄 needs_review (ignore any code/closed PRs)

**Scenario C: MERGED PR + CLOSED PR + no code**
- MERGED PR found → check code → no code found
- Result: ✅ implemented (low confidence - PR only)
- Note: CLOSED PR is ignored

**Scenario D: CLOSED PR + code found**
- No OPEN/MERGED PRs → check code → code found
- Result: ✅ implemented (medium confidence - code only)
- Note: CLOSED PR doesn't trigger needs_review

**Scenario E: CLOSED PR + no code**
- No OPEN/MERGED PRs, no code/config
- Result: ❌ not_implemented
- Note: CLOSED PR means abandoned attempt, feature not implemented

## Path Filtering Rules

**Critical for accuracy in shared repositories:**

- Always use path filters when provided
- Android and Java share `getsentry/sentry-java` but have separate codebases
- Without path filters, searches return identical results for both SDKs
- Code in `sentry-android/` does NOT mean Java SDK has the feature
- Code in `sentry/` does NOT mean Android SDK has the feature

**Substring matching protection:**
- GitHub's `path:` qualifier may perform substring matching
- Use negative filters to prevent false matches: `-path:sentry-android/`
- Example: `path:sentry/ -path:sentry-android/` ensures Java-only results
