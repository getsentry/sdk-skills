# Pattern Extraction Algorithm

Detailed algorithm for extracting search patterns from a reference PR.

## Input

- PR title: String
- PR body: Markdown text
- PR files: Array of file paths

## Output

```json
{
  "keywords": ["client reports", "client reporting"],
  "code_patterns": ["ClientReport", "ClientReportManager", "client_report"],
  "config_options": ["sendClientReports", "send_client_reports"]
}
```

## Algorithm

### A. Extract Feature Name from Title

Remove noise from PR title to get feature name:

1. Remove prefixes: `feat:`, `Add `, `Implement `, `feature:`, `fix:`
2. Remove SDK references: `to Python SDK`, `for iOS`, `(python)`
3. Remove suffixes: ` support`, ` feature`

**Examples:**
- "feat(python): Add Client Reports support" → "Client Reports"
- "Implement Session Replay for iOS" → "Session Replay"

### B. Extract Code Patterns from Body

1. **Extract code blocks**: Get text from triple backticks and inline code
2. **Apply regex to code blocks** (line by line):
   - Classes: `[A-Z][a-zA-Z0-9]+`
   - Functions: `[a-z][a-zA-Z0-9]+\(`
3. **Filter noise**: Remove `String`, `Boolean`, `Integer`, `Object`, `Array`, `List`, `Map`, `Set`, `Dict`, `HashMap`, `ArrayList`, `Optional`, `Result`, `Promise`, `Error`, `Exception`, `Response`, `Request`, `Any`, `Void`, `Null`
4. **Extract config keys**: Look for JSON/YAML patterns `"key": value`

### C. Extract Patterns from File Paths

1. Take filename without extension
2. Skip test/docs/config files
3. Limit to top 10 files by path length (shorter = core implementation)

### D. Generate Language Variants

For each base pattern, generate naming convention variants:

| Convention | Example | With Suffix |
|------------|---------|-------------|
| CamelCase | `ClientReport` | `ClientReportManager` |
| camelCase | `clientReport` | `clientReportManager` |
| snake_case | `client_report` | `client_report_manager` |
| kebab-case | `client-report` | `client-report-manager` |

Common suffixes: `Manager`, `Recorder`, `Handler`, `Service`, `Reporter`, `s`

### E. Frequency Counting

Count pattern occurrences across all sources:

- **PR title**: +5 points per occurrence
- **PR body**: +1 point per occurrence (case-insensitive)
- **File paths**: +3 points per occurrence

Sort by total points (descending), break ties alphabetically.

### F. Categorize and Select

1. Organize into keywords/code_patterns/config_options
2. Select top 5 code_patterns by points
3. Select top 3 config_options by points
4. Include all keywords

### G. Fallback Strategy

If extraction yields <3 total patterns:
1. Use feature name from title as keyword
2. Generate CamelCase, snake_case, kebab-case variants
3. Continue with these patterns
