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

See [references/sdk-mappings.md](references/sdk-mappings.md) for complete SDK to repository mapping (17 SDKs total).

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

For each SDK in the mapping, check implementation status using a **three-stage detection approach**:

#### Stage 1: Search for Develop Doc References

```bash
# Search PRs mentioning develop doc
gh search prs --repo <SDK_REPO> --match body "<develop-doc-url>" --limit 5

# Search commits mentioning develop doc
gh search commits --repo <SDK_REPO> "<develop-doc-title>" --limit 5
```

#### Stage 2: Search Git History

```bash
# Search for PRs with related titles
gh search prs --repo <SDK_REPO> --match title "<feature-name>" --limit 5
gh search prs --repo <SDK_REPO> --match title "<config-option>" --limit 5

# Check PR states
for each found PR:
  gh pr view <PR_NUM> --repo <SDK_REPO> --json state,mergedAt,url
```

#### Stage 3: Search Code Patterns

```bash
# Search for configuration options from develop doc
gh search code "<config-option-name>" --repo <SDK_REPO>

# Search for class/function names from reference implementation
gh search code "<class-name>" --repo <SDK_REPO>
gh search code "<function-name>" --repo <SDK_REPO>
```

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

**`.sdk-align/context.json`** - Feature metadata including name, develop doc details, reference implementation, alignment scope, config options, and Linear issue.

**`.sdk-align/status-report.json`** - Status for each SDK including implementation state, PR links, merge dates, and notes.

Inform user that context has been saved for other `sdk-*` skills.

## Output

Display to user:
1. Formatted status table
2. Summary statistics
3. List of SDKs needing implementation
4. Location of saved context files
5. Suggested next steps (e.g., "Run /sdk-feature-generate go to implement for Go SDK")

## Guidelines

### Progress Tracking

**Use TodoWrite to track the status check process:**

Create a todo list with the following steps:
1. Gather develop doc
2. Find reference implementation
3. Check all SDKs for implementation status
4. Generate status report
5. Write shared context files

Mark each step as in_progress when starting and completed when finished. When checking SDKs, consider adding individual todos for each SDK or SDK group to track progress through the ~19 SDK repositories.

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

### Performance Optimization

- Run SDK checks in parallel where possible (use background processes)
- Cache GitHub API responses
- Limit search results to recent PRs (last 2 years)
- Skip code search for SDKs already found via PR/commit search

