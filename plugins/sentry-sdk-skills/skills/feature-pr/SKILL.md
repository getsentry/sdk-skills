---
name: sdk-feature-pr
description: Create GitHub pull request for a Sentry SDK. Use when creating PR, pull request, or merging SDK code. Keywords: PR, pull request, create PR, SDK, github.
argument-hint: [sdk-name]
allowed-tools: Read Write Bash Skill AskUserQuestion
compatibility: Requires gh CLI and sentry-skills:create-pr.
---

# SDK Feature PR Creator

Context adapter that creates PRs for SDK implementations. Reads alignment context, invokes sentry-skills:create-pr, tracks results, and updates Linear.

## When to Use This Skill

Use this skill when:
- You've implemented a feature in an SDK (manually or via `sdk-feature-generate`)
- You need to create a PR with proper links to develop docs, reference PR, and Linear issue
- You want to retry PR creation after a failure
- You deferred PR creation during generation

## Shared Context System

This skill:
- **Reads** from `.sdk-align/context.json` (develop doc, reference impl, Linear issue)
- **Reads** from `.sdk-align/implementations.json` (branch name, commit info, repo path)
- **Writes** to `.sdk-align/prs.json` (track created PRs)
- **Works standalone** if all information is provided directly

## SDK to Repository Mapping

See [../../references/sdk-mappings.md](../../references/sdk-mappings.md) for complete SDK to repository mapping (17 SDKs).

## Process

### Step 1: Gather Context

**Required input:**
- Target SDK (e.g., "python", "go")

**Important**: Save the current working directory path for later use - all `.sdk-align/` file operations should use this directory, not the SDK repository directory.

**Read from shared context:**

Use the Read tool to check for and read (in current directory):
- `.sdk-align/context.json` (feature context: develop doc, reference PR, Linear issue)
- `.sdk-align/implementations.json` (implementation details: branch, repo path, feature name)
- `.sdk-align/prs.json` (check if PR already created)

**If context files don't exist:**
Ask user for:
1. Branch name (required)
2. Repository path (local or /tmp/)
3. Develop doc URL (recommended for context)
4. Reference implementation PR URL (recommended for context)
5. Linear issue ID (optional)

### Step 2: Navigate to Repository

Navigate to repository:
- `cd <repo-path>` (from implementations.json or user input)

Verify and switch to correct branch:
- Check current branch: `git branch --show-current`
- If not on expected branch: `git checkout <branch-name>`
- If branch doesn't exist: Error - implementation branch not found, check branch name

### Step 3: Invoke sentry-skills:create-pr

Invoke the create-pr skill with SDK alignment context:
```
Skill(sentry-skills:create-pr)
```

**When create-pr prompts, provide:**
- Develop doc URL (for feature details in PR description)
- Reference implementation PR URL (for implementation context)
- Linear issue ID (for automatic linking)
- SDK-specific implementation notes or deviations

This enriches the PR description with SDK alignment context.

**If sentry-skills:create-pr is not available:**
Display message: "The sentry-skills plugin with create-pr skill is required. Please install it or manually create PR with `gh pr create`."

### Step 4: Track Created PR

After PR is created successfully, write details to `.sdk-align/prs.json` in the original working directory (saved in Step 1):

```json
{
  "feature": "<feature-name>",
  "prs": [
    {
      "sdk": "<sdk-name>",
      "pr_url": "<github-pr-url>",
      "pr_number": <number>,
      "branch": "<branch-name>",
      "state": "open",
      "created_at": "<iso-timestamp>",
      "links": {
        "develop_doc": "<develop-doc-url>",
        "reference_pr": "<reference-pr-url>",
        "linear_issue": "<linear-issue-url>"
      }
    }
  ]
}
```

**Use Write tool to update or create this file:**
- If file doesn't exist: Write new file with single-item array
- If file exists: Read it, parse JSON, append to `prs` array, write back

### Step 5: Update Linear Issue (Optional)

If Linear issue ID is available, add PR link to the issue:

**Using Linear MCP (if available):**
- Add comment to Linear issue with PR link
- Optionally update issue status (e.g., "In Review")

**Manual tracking:**
- Inform user to update Linear issue manually
- Provide suggested comment text with PR link

### Step 6: Output Summary

```
✅ PR created: <url>
Branch: <branch-name> | PR #<number>
Tracked in .sdk-align/prs.json
```

## Edge Cases

**PR already exists:**
- Check `.sdk-align/prs.json` first
- Query GitHub: `gh pr list --head <BRANCH>`
- Inform user with existing PR URL
- Offer to update tracking if not recorded

**Context files missing:**
- Ask user for required information interactively
- Can still create PR without full context
- Won't be able to track in prs.json without context

**Delegated to create-pr:**
The following are handled by sentry-skills:create-pr:
- Branch not pushed to remote (will prompt or auto-push)
- SDK repo not accessible (will suggest `gh auth login`)
- No commits on branch (will inform user to commit first)
- PR template merging
- Branch state verification

