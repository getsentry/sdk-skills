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

- ✅ **implemented**: Merged PR + code evidence found IN THE CORRECT PATH
  - Set confidence based on evidence strength (see Output section)
- ⚠️ **needs_review**: Open PR or unclear implementation (confidence = null)
- ❌ **not_implemented**: No evidence found IN THE CORRECT PATH (confidence = null)
- 🚫 **not_applicable**: Feature doesn't apply to this SDK type (confidence = null)
- ⚠️ **error**: gh command failed (confidence = null, include error message)

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
