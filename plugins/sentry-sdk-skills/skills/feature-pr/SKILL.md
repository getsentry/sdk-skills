---
name: sdk-feature-pr
description: Create pull request for ONE Sentry SDK with proper links to develop docs, reference implementation, and Linear issue. Uses sentry-skills:create-pr for Sentry conventions. Works standalone or with context from other sdk-* skills.
model: sonnet
allowed-tools: Read Write Bash Skill AskUserQuestion
compatibility: Requires gh CLI and sentry-skills:create-pr skill.
---

# SDK Feature PR Creator

Create a pull request for a Sentry SDK implementation with proper links and structure.

## When to Use This Skill

Use this skill when:
- You've implemented a feature in an SDK (manually or via `sdk-feature-generate`)
- You need to create a PR with proper links to develop docs, reference PR, and Linear issue
- You want to follow Sentry PR conventions automatically

## Shared Context System

This skill can:
- **Read** from `.sdk-align/context.json` (develop doc, reference impl, Linear issue)
- **Read** from `.sdk-align/implementations.json` (branch name, commit info)
- **Write** to `.sdk-align/prs.json` (track created PRs)
- **Work standalone** if all information is provided directly

## SDK to Repository Mapping

- `python` → `getsentry/sentry-python`
- `javascript` → `getsentry/sentry-javascript`
- `ruby` → `getsentry/sentry-ruby`
- `php` → `getsentry/sentry-php`
- `go` → `getsentry/sentry-go`
- `java` → `getsentry/sentry-java`
- `dotnet` → `getsentry/sentry-dotnet`
- `rust` → `getsentry/sentry-rust`
- `android` → `getsentry/sentry-java`
- `cocoa` → `getsentry/sentry-cocoa`
- `react-native` → `getsentry/sentry-react-native`
- `flutter` → `getsentry/sentry-dart`
- `unity` → `getsentry/sentry-unity`
- `unreal` → `getsentry/sentry-unreal`
- `native` → `getsentry/sentry-native`
- `elixir` → `getsentry/sentry-elixir`

## Process

### Step 1: Gather Required Information

**Required inputs:**
- Target SDK (e.g., "python", "go")

**Optional inputs (will be read from context if not provided):**
- Branch name
- Develop doc URL
- Reference implementation PR URL
- Linear issue ID/URL
- Feature name

**Read from shared context:**

Use the Read tool to check for and read:
- `.sdk-align/context.json` (feature context)
- `.sdk-align/implementations.json` (implementation details)
- `.sdk-align/prs.json` (check if PR already created)

**If context files don't exist:**
Ask user for:
1. Branch name (required)
2. Develop doc URL (required)
3. Reference implementation PR URL (optional but recommended)
4. Linear issue ID (optional)

### Step 2: Verify Branch State

Check that the branch exists and is pushed to remote:

```bash
# Get SDK repo
REPO=$(get_repo_for_sdk <SDK>)

# Check if branch exists on remote
gh api repos/$REPO/branches/<BRANCH_NAME> --jq '.name'
```

**If branch doesn't exist on remote:**
- Check if it exists locally in implementations.json
- If `pushed: false`, ask user if they want to push it
- If yes: `git push -u origin <branch-name>`
- If no: Cannot create PR, branch must be pushed first

**Get branch details:**
```bash
# Get latest commit on branch
gh api repos/$REPO/branches/<BRANCH_NAME> --jq '.commit.sha'

# Get commit details
gh api repos/$REPO/commits/<COMMIT_SHA> --jq '{message: .commit.message, author: .commit.author.name}'
```

### Step 3: Build PR Title

Follow Sentry conventions: `<type>(<scope>): <description>`

- **Type**: feat, fix, ref, test, docs
- **Scope**: SDK name (python, go, javascript, etc.)
- **Description**: Imperative mood ("Implement" not "Implements")

Example: `feat(python): Implement strict trace continuation`

### Step 4: Build PR Body

Create PR description following this structure:

```markdown
## Summary

<One-line summary of what this PR does>

<Paragraph explaining the feature and why it's being implemented>

## Implementation Details

<Bullet points describing key changes>
- Implements <detail-1>
- Adds <detail-2>
- Updates <detail-3>

## References

- **Develop Doc**: <develop-doc-url>
- **Reference Implementation**: <reference-pr-url>
- **Linear Issue**: <linear-issue-url>
- **Alignment Scope**: <Core|Domain|Platform>

## Testing

<Brief description of tests added/updated>

- <test-type-1>: <description>
- <test-type-2>: <description>

## SDK-Specific Notes

<Any deviations from reference implementation or SDK-specific considerations>
```

### Step 5: Invoke sentry-skills:create-pr

Use the existing `sentry-skills:create-pr` skill to create the PR with Sentry conventions:

**Before invoking create-pr:**

Ensure branch is pushed and all changes are committed:
```bash
# Check for uncommitted changes in the branch
gh api repos/$REPO/compare/main...<BRANCH_NAME> --jq '.files | length'
```

**Invoke the skill:**
```bash
# Change to SDK repo directory (if local)
cd /tmp/<sdk-name>

# Invoke create-pr skill
Skill(sentry-skills:create-pr)
```

The create-pr skill will:
- Generate properly formatted PR title
- Use the PR body we prepared
- Handle any SDK-specific PR templates
- Create the PR via `gh pr create`
- Return the PR URL

**Alternative: Direct gh pr create (if create-pr skill not available):**

```bash
gh pr create \
  --repo <REPO> \
  --head <BRANCH_NAME> \
  --title "<PR_TITLE>" \
  --body "$(cat <<'EOF'
<PR_BODY>
EOF
)"
```

### Step 6: Track Created PR

Write PR details to `.sdk-align/prs.json`:

```json
{
  "version": "1.0",
  "prs": {
    "python": {
      "sdk": "python",
      "feature": "Strict Trace Propagation",
      "pr": "https://github.com/getsentry/sentry-python/pull/9999",
      "prNumber": 9999,
      "branch": "feat/strict-trace-continuation",
      "state": "open",
      "createdAt": "2026-01-18T00:30:00Z",
      "developDoc": "https://github.com/getsentry/sentry-docs/pull/11912",
      "referenceImplementation": "https://github.com/getsentry/sentry-javascript/pull/16313",
      "linearIssue": "GSD-1234"
    }
  }
}
```

Update or create this file with the new PR.

### Step 7: Update Linear Issue (Optional)

If Linear issue ID is available, add PR link to the issue:

**Using Linear MCP (if available):**
- Add comment to Linear issue with PR link
- Update issue status if needed (e.g., "In Review")

**Manual tracking:**
- Inform user to update Linear issue manually
- Provide suggested comment text

### Step 8: Output Summary

Display to user:
- ✅ PR created for `<sdk>`
- PR URL: `<url>`
- PR number: `#<number>`
- Branch: `<branch-name>`
- Linked to develop doc: ✓
- Linked to reference PR: ✓
- Linked to Linear issue: ✓ (if applicable)
- Context saved to `.sdk-align/prs.json`

Show the PR URL prominently so user can click to view.

## Guidelines

### PR Description Best Practices

1. **Clear Summary**
   - First sentence: What does this PR do?
   - Second paragraph: Why (motivation, context)
   - Be specific, avoid vague language

2. **Implementation Details**
   - Bullet points, easy to scan
   - Highlight key changes, not every file
   - Mention significant architectural decisions

3. **Always Include References**
   - Develop doc is the source of truth
   - Reference implementation shows the pattern
   - Linear issue for tracking
   - Alignment scope helps reviewers understand requirements

4. **Testing Section**
   - Don't just list "added tests"
   - Describe what scenarios are covered
   - Mention if integration tests are included

5. **SDK-Specific Notes**
   - Call out any deviations from reference
   - Explain platform-specific constraints
   - Note compatibility considerations

### Handling Edge Cases

**Branch not pushed:**
- Ask user if they want to push
- Show command to push manually
- Cannot create PR without pushed branch

**PR already exists for branch:**
- Check `.sdk-align/prs.json` first
- Query GitHub: `gh pr list --repo <REPO> --head <BRANCH>`
- Offer to update existing PR description
- Or inform user PR already exists

**SDK repo not accessible:**
- Check `gh auth status`
- Suggest `gh auth login`
- Cannot proceed without access

**No commit on branch:**
- Verify branch has commits different from main
- If branch is empty, inform user to commit changes first

**PR template exists:**
- Fetch template: `gh repo view --json pullRequestTemplates`
- Merge template requirements with generated body
- Ensure all template sections are filled

### Multi-SDK PR Creation

If user wants to create PRs for multiple SDKs at once:

```bash
# User: /sdk-feature-pr python,go,java
```

Process each SDK sequentially:
1. Read implementation for each SDK
2. Verify branch state for each
3. Create PR for each
4. Track all PRs
5. Display summary table

**Summary output for multiple PRs:**
```
PRs Created:
────────────────────────────────────────────────────────────
SDK         PR Number    Status    URL
────────────────────────────────────────────────────────────
python      #9999        ✅ Open   https://github.com/.../9999
go          #8888        ✅ Open   https://github.com/.../8888
java        #7777        ✅ Open   https://github.com/.../7777
────────────────────────────────────────────────────────────
```

