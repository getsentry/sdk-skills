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
gh search prs --repo {repo} "{keywords}" --limit 10 --json number,title,state,url,mergedAt
```

### 2. Code Search

**CRITICAL**: Include path filter to avoid false matches in shared repositories.

- **If path_filter is provided**:
  ```bash
  gh search code --repo {repo} "{code_pattern} {path_filter}" --json path,repository
  ```

- **If no path_filter**:
  ```bash
  gh search code --repo {repo} "{code_pattern}" --json path,repository
  ```

**Examples for shared repo:**
```bash
# Android in getsentry/sentry-java
gh search code --repo getsentry/sentry-java "ClientReport path:sentry-android/" --json path,repository

# Java in getsentry/sentry-java (with negative filter to exclude Android)
gh search code --repo getsentry/sentry-java "ClientReport path:sentry/ -path:sentry-android/" --json path,repository
```

**IMPORTANT**: When path_filter is provided, ONLY count code matches within that path.
- For android (`path:sentry-android/`): ONLY files in `sentry-android/` directory
- For java (`path:sentry/ -path:sentry-android/`): ONLY files in `sentry/` directory, explicitly excluding `sentry-android/`
- The negative filter `-path:sentry-android/` is CRITICAL to prevent false positives from substring matching

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
  "pr_url": "https://..." or null,
  "pr_number": 1234 or null,
  "merged_date": "2024-01-15" or null,
  "notes": "Brief finding (1 sentence, mention path if filtered)",
  "error": null or "error message"
}
```

## Status Determination

- ✅ **implemented**: Merged PR + code evidence found IN THE CORRECT PATH
- ⚠️ **needs_review**: Open PR or unclear implementation
- ❌ **not_implemented**: No evidence found IN THE CORRECT PATH
- 🚫 **not_applicable**: Feature doesn't apply to this SDK type
- ⚠️ **error**: gh command failed (include error message)

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
