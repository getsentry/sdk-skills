---
name: sdk-feature-status
description: Check which Sentry SDKs have a feature and which need it. Use when auditing SDK status, checking implementation progress, or starting SDK alignment. Keywords: check status, which SDKs, SDK alignment, feature status.
argument-hint: [develop-doc-url]
allowed-tools: Read Grep Glob Bash Write AskUserQuestion TodoWrite
compatibility: Requires gh CLI with authentication.
---

# SDK Feature Status Checker

Check implementation status of a feature across all Sentry SDKs based on develop docs and reference implementations.

## When to Use This Skill

Use this skill when:
- You want to audit which SDKs have implemented a feature
- You're starting SDK alignment work and need to understand the landscape
- You need to find the reference implementation for a feature
- You want to create shared context for other `sdk-*` skills

## Shared Context System

This skill creates/updates `.sdk-align/` directory with shared context:

```
.sdk-align/
├── context.json          # Feature context (develop doc, reference impl, Linear issue)
└── status-report.json    # Implementation status per SDK
```

Other `sdk-*` skills can read this context to work together seamlessly.

## SDK to Repository Mapping

See [references/sdk-mappings.md](../../references/sdk-mappings.md) for complete SDK to repository mapping (17 SDKs total).

## Process

### Step 1: Gather Develop Doc

Ask user for the develop doc. Accept:
- **PR URL**: `https://github.com/getsentry/sentry-docs/pull/XXXX` or `https://github.com/getsentry/develop/pull/XXXX`
- **Commit SHA**: From either sentry-docs or develop repo
- **Direct URL**: Link to develop.sentry.dev
- **File path**: Local path to develop doc MDX file

Fetch the develop doc:

```bash
# If PR URL provided
gh pr view <PR_NUM> --repo <REPO> --json title,body,files,state,url

# If commit SHA provided
gh api repos/<REPO>/commits/<SHA>
```

Analyze the develop doc to extract:
- Feature name
- Alignment scope (Core/Domain/Platform)
- Key requirements (MUST/SHOULD/MAY)
- Configuration options
- Public API details

### Step 2: Find Reference Implementation

**If user provides reference implementation:**
- SDK name and optional PR URL
- Fetch PR details

**If not provided, search for it:**
- Search for PRs mentioning the develop doc PR number using `gh search prs`
- Search for PRs with related titles
- Search code for feature-specific identifiers using `gh search code`
- Filter results to SDK repos only (exclude sentry-docs, sentry, etc.)
- Find the earliest merged PR as reference

Extract: SDK name, PR URL, merge date, key patterns

### Step 3: Check All SDKs

**Use TodoWrite to track progress:**

Create a todo list with each SDK to check:
```
1. Check JavaScript SDK (reference)
2. Check Python SDK
3. Check Ruby SDK
...
17. Check Unity SDK
```

Mark each as in_progress while checking, completed when done.

For each SDK in the mapping, check implementation status using three-stage detection:

1. **Search for develop doc references** - PRs/commits mentioning develop doc URL or title
2. **Search git history** - PRs with feature name or config options in title
3. **Search code patterns** - Config options, class names, function names from reference

#### Categorize Status

For each SDK, determine one of:
- ✅ **Implemented** - Feature fully implemented, PR merged
- ⚠️ **Needs Review** - Partial implementation or open PR
- ❌ **Not Implemented** - Feature missing entirely
- 🚫 **Not Applicable** - Feature doesn't apply to this SDK (based on alignment scope)

Collect metadata:
- PR URL (if exists)
- Merge date (if merged)
- Implementation notes
- Deviation from spec (if detected)

### Step 4: Smart Filtering by Alignment Scope

Based on the develop doc's alignment scope, filter applicable SDKs:

**Core Scope:**
- All SDKs with distributed tracing support (typically all SDKs)

**Domain Scope:**
- **Backend**: python, ruby, php, go, java, dotnet, elixir, rust
- **Frontend**: javascript (browser), react-native
- **Mobile**: android, cocoa, flutter, react-native
- **Universal**: Features that apply across domains

**Platform Scope:**
- Feature may vary by platform
- Check if spec defines platform-specific requirements

Mark SDKs as "Not Applicable" if they don't match the scope.

### Step 5: Generate Status Report

Create a formatted status table showing:
- Feature name, develop doc URL, reference SDK/PR, alignment scope
- Implementation status for each SDK (✅ Implemented, ⚠️ Needs Review, ❌ Not Implemented, 🚫 Not Applicable)
- Summary counts
- List of SDKs needing implementation

Display this table to the user.

### Step 6: Write Shared Context

Create `.sdk-align/` directory and write context files:

**`.sdk-align/context.json`**:
```json
{
  "feature": "<feature-name>",
  "develop_doc": {
    "url": "<pr-or-commit-url>",
    "type": "pr|commit|file"
  },
  "reference_implementation": {
    "sdk": "<reference-sdk-name>",
    "pr_url": "<pr-url>",
    "pr_number": <number>
  },
  "alignment_scope": {
    "domain": ["backend", "frontend", "mobile", "universal"],
    "sdks": ["python", "go", ...]
  },
  "linear_issue": "<issue-id-or-url>",
  "timestamp": "<iso-timestamp>"
}
```

**`.sdk-align/status-report.json`**:
```json
{
  "feature": "<feature-name>",
  "sdks": [
    {
      "name": "<sdk-name>",
      "status": "implemented|needs_review|not_implemented|not_applicable",
      "pr_url": "<url-if-found>",
      "pr_number": <number-if-found>,
      "merge_date": "<iso-date-if-merged>",
      "notes": "<any-notes>"
    }
  ],
  "summary": {
    "implemented": <count>,
    "needs_review": <count>,
    "not_implemented": <count>,
    "not_applicable": <count>
  }
}
```

**Use Write tool to create both files:**
- Create `.sdk-align/` directory in current working directory (if needed)
- Write `.sdk-align/context.json`
- Write `.sdk-align/status-report.json`

**Important**: The `.sdk-align/` directory is created in the current working directory. Other skills will look for it in the same location, so run all SDK alignment commands from the same directory.

Inform user that context has been saved for other `sdk-*` skills.

## Output

Display to user:
1. Formatted status table
2. Summary statistics
3. List of SDKs needing implementation
4. Location of saved context files

**Suggested next steps:**
- For single SDK: `/sdk-feature-generate {sdk-name}` (e.g., `/sdk-feature-generate python`)
- For multiple SDKs: Run `/sdk-feature-generate` for each SDK sequentially
- For Linear tracking: `/linear-initiative {feature-name}` (creates projects for SDK teams)

Context saved to `.sdk-align/` - other skills will automatically use this.

## Guidelines

### Progress Tracking

**Use TodoWrite throughout:**

Create main workflow todos:
1. Gather develop doc
2. Find reference implementation
3. Check all SDKs for implementation status (see Step 3 for per-SDK tracking)
4. Generate status report
5. Write shared context files

In Step 3, create individual todos for each SDK to provide real-time progress as checking takes time.

### Handling Edge Cases

**Develop doc not accessible:**
- Ask user to provide develop doc content or file path
- If commit SHA, fetch raw file content via GitHub API

**Reference implementation not found:**
- Ask user which SDK should be reference
- Or continue without reference (use only develop doc)

**SDK repo not accessible:**
- Check `gh auth status`
- Skip that SDK and mark as "unknown"
- Continue with accessible SDKs

**Multiple implementations found:**
- Choose the earliest merged PR as reference
- List other implementations in notes

**Conflicting implementations:**
- Flag SDKs that have different implementations
- Note in status report for manual review

