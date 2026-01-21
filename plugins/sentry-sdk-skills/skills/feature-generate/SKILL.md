---
name: sdk-feature-generate
description: Generate code for a Sentry SDK. Use when implementing a feature in python, javascript, go, ruby, java, or other SDKs. Keywords: implement, generate, code, SDK, feature.
argument-hint: [sdk-name]
allowed-tools: Read Grep Glob Bash Write Edit Task TodoWrite Skill AskUserQuestion
compatibility: Requires gh CLI, git, SDK-specific linting tools, and sentry-skills:commit.
---

# SDK Feature Code Generator

Generate idiomatic implementation code for a specific Sentry SDK based on develop docs and reference implementation.

## When to Use This Skill

Use this skill when:
- You need to implement a feature in a specific SDK
- You have a develop doc and reference implementation
- You want idiomatic, linted code following SDK conventions
- You're working through SDK alignment one SDK at a time

## Planning Complex Implementations

For complex or unfamiliar features, consider using EnterPlanMode before generating code:

**Use EnterPlanMode when:**
- The feature involves significant architectural changes
- You're unfamiliar with the target SDK's codebase structure
- Multiple approaches are possible and you need to design the best one
- The feature touches many files or subsystems

**In plan mode, you can:**
- Thoroughly explore the target SDK's codebase
- Study similar existing features
- Design the implementation approach
- Identify all files that need changes
- Get user approval on the approach before writing code

**Skip plan mode when:**
- The feature is straightforward and well-defined
- You've already explored the SDK in a previous step
- The reference implementation provides a clear pattern to follow

## Shared Context System

This skill can:
- **Read** from `.sdk-align/context.json` (created by `sdk-feature-status`)
- **Write** to `.sdk-align/implementations.json` (tracks generated implementations)
- **Work standalone** if develop doc and reference are provided directly

## SDK to Repository Mapping

See [../../references/sdk-mappings.md](../../references/sdk-mappings.md) for complete SDK to repository mapping (17 SDKs).

## Process

### Step 1: Gather Required Information

**Required inputs:**
- Target SDK (e.g., "python", "go", "java")
- Develop doc (PR/commit/URL) OR read from `.sdk-align/context.json`
- Reference implementation (SDK + PR) OR read from context

**Optional inputs:**
- Branch name (auto-generates if not provided: `feat/<feature-slug>`)
- Push to remote (default: false, just commit locally)

**Important**: Save the current working directory path for later use - all `.sdk-align/` file operations should use this directory, not the SDK repository directory.

**If context file exists:**

Use the Read tool to check for and read `.sdk-align/context.json` in current directory.

Extract feature name, develop doc, and reference implementation from context.

**If no context file:**
Ask user for:
1. Develop doc URL/path
2. Reference SDK and PR URL

### Step 2: Fetch Reference Implementation

Clone or fetch the reference SDK repository:

```bash
# Clone reference SDK
gh repo clone <REFERENCE_SDK_REPO> /tmp/reference-sdk -- --depth 1

# Fetch the reference PR files
gh pr view <PR_NUM> --repo <REFERENCE_SDK_REPO> --json files --jq '.files[] | {filename, additions, deletions}'
```

Analyze reference implementation:
- **Core files changed** (implementation, config, utils)
- **Test files added/changed**
- **Code patterns** (class names, function signatures, variable names)
- **Configuration options** (new config fields, defaults)
- **Documentation comments** (docstrings, JSDoc)

### Step 3: Locate or Clone Target SDK Repository

**Ask user**: "Do you have a local checkout of {SDK}?"

**If yes (local workflow):**
- Ask for repository path (e.g., `~/code/sentry-python`)
- Convert to absolute path using `realpath` or similar
- Validate it's the correct repository using `git remote -v`
- Check current branch with `git branch --show-current`
- If not on main/master, warn: "Currently on {branch}. Should I create feature branch from here or switch to main first?"
- If uncommitted changes exist, offer to stash them
- Create feature branch: `git checkout -b <branch-name>`

**If no (automated workflow):**
- Clone to `/tmp/<target-sdk-name>` using `gh repo clone <TARGET_SDK_REPO> /tmp/<target-sdk-name> -- --depth 1`
- **Warn user**: "Working in temporary directory. Changes will be pushed automatically to avoid loss."
- Create feature branch: `git checkout -b <branch-name>`

**Important**: Always store the absolute path in implementations.json for reliable navigation.

**Branch naming:**
- Suggest: `feat/<feature-slug>` (e.g., `feat/client-reports`)
- User can override with custom name

### Step 4: Analyze Target SDK Patterns

Study the target SDK to understand conventions:
- Explore directory structure to find where similar features are implemented
- Identify naming conventions (use Glob/Grep on existing files)
- Locate test framework and patterns (check test directories)
- Find linting configuration files (.eslintrc, pyproject.toml, .rubocop.yml, etc.)

### Step 5: Generate Idiomatic Implementation

Generate code adapting reference implementation to target SDK conventions:

**Core implementation:**
- Follow SDK naming patterns (snake_case vs camelCase, etc.)
- Use SDK type system appropriately
- Place files in conventional locations
- Add documentation in SDK style

**Configuration:**
- Add options following SDK config patterns
- Set appropriate defaults

**Tests:**
- Use SDK test framework
- Match SDK test naming conventions
- Mirror reference implementation coverage
- Use SDK-specific fixtures/helpers

### Step 6: Run Linting and Tests

Run formatters, linters, type checking (if used), and test suite.

**If linting/tests fail**, ask: "What would you like to do?"
- Try auto-fix
- Continue anyway (note failures in context)
- Let me fix manually
- Skip this SDK

### Step 7: Review and Commit Changes

**Review (optional for local workflow):**
Ask: "Code generated. What would you like to do?"
- Show me the diff (concise: files, key changes, tests)
- Continue to commit
- Let me review manually

**Commit:**
Stage changes: `git add .`
Invoke: `Skill(sentry-skills:commit)` (handles Sentry conventional format)

### Step 8: Push Branch

**If working in /tmp/ (automated workflow):**
- **Always push** to avoid data loss: `git push -u origin <branch-name>`
- Inform user: "Pushed to remote. Branch: {branch-name}"

**If working in local checkout:**
Use AskUserQuestion: "Push branch to remote?"

Options:
1. "Yes, push now" - Run `git push -u origin <branch-name>`
2. "No, I'll push later" - Note in implementations.json: `pushed: false`

If not pushed, warn: "Remember to push before creating PR with /sdk-feature-pr"

### Step 9: Update Implementations Context

**Note**: Ensure writing to `.sdk-align/implementations.json` in the original working directory (where feature-status was run), not the SDK repository directory.

Write implementation details to `.sdk-align/implementations.json`:

```json
{
  "feature": "<feature-name>",
  "implementations": [
    {
      "sdk": "<sdk-name>",
      "branch": "<branch-name>",
      "repo_path": "<local-path-or-/tmp/path>",
      "commit_sha": "<sha>",
      "files_changed": <count>,
      "pushed": <true/false>,
      "linting_passed": <true/false>,
      "tests_passed": <true/false>,
      "timestamp": "<iso-timestamp>"
    }
  ]
}
```

**Use Write tool to update or create this file:**
- If file doesn't exist: Write new file with single-item array
- If file exists: Read it, parse JSON, append to `implementations` array, write back

### Step 10: Create Pull Request (Optional)

Use AskUserQuestion: "Create pull request now?"

Options:
1. "Yes, create PR" - Invoke sdk-feature-pr to create PR with tracked context
2. "No, I'll create it later" - Skip, user can run /sdk-feature-pr later

**If yes:**
- Invoke `Skill(sdk-feature-pr, args: "<sdk-name>")`
- sdk-feature-pr will read context, create PR, track results, and update Linear
- Display PR URL in summary

**If no:**
- Inform user: "You can create a PR later by running `/sdk-feature-pr <sdk>`"
- All context is saved and ready for deferred PR creation

### Step 11: Output Summary

```
✅ <sdk> implementation complete
Branch: <branch-name> | <commit-sha> | <count> files
Linting: ✓ | Tests: ✓ (<X> tests) | Pushed: <yes/no>

[If PR created:]
PR: <url>

[If deferred:]
Next: /sdk-feature-pr <sdk>
```

## Guidelines

- Use TodoWrite to track the 10-step implementation process
- Stay idiomatic - don't translate code literally, use language-specific patterns
- Match existing patterns in target SDK (naming, structure, tests)
- Respect type systems appropriate to each SDK
- Complete implementation: core functionality, config, tests, docs, error handling


