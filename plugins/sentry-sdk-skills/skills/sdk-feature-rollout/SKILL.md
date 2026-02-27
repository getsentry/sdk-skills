---
name: sdk-feature-rollout
description: Roll out an SDK feature across multiple Sentry SDK repositories. Use when implementing a spec across SDKs, creating GitHub issues for SDK repos, or spawning parallel implementation agents. Triggers on "rollout", "implement across SDKs", "SDK feature rollout", "cross-SDK implementation".
allowed-tools: Bash Read Grep Glob WebFetch Task AskUserQuestion
compatibility: Requires `gh` CLI authenticated with GitHub. Optionally requires the Linear MCP server for initiative tracking.
---

# SDK Feature Rollout

Automates rolling out a feature from spec to implementation across multiple Sentry SDK repositories.

## SDK Repos

All repos are under the `getsentry` GitHub org. Only repos with Enabled=Yes are included by default.

| Repo | Suffix | Category | Enabled |
|------|--------|----------|---------|
| sentry-android | Android | Mobile | Yes |
| sentry-capacitor | Capacitor | JavaScript, Mobile | Yes |
| sentry-cocoa | Apple | Mobile | Yes |
| sentry-cordova | Cordova | Mobile | Yes |
| sentry-dart | Dart/Flutter | Mobile | Yes |
| sentry-dotnet | .NET | Backend | Yes |
| sentry-electron | Electron | JavaScript | Yes |
| sentry-elixir | Elixir | Backend | Yes |
| sentry-go | Go | Backend | Yes |
| sentry-godot | Godot | Gaming | Yes |
| sentry-java | Java | Backend | Yes |
| sentry-javascript | JavaScript | JavaScript | Yes |
| sentry-kotlin-multiplatform | KMP | Mobile | Yes |
| sentry-laravel | Laravel | Backend | Yes |
| sentry-lynx | Lynx | Mobile | No |
| sentry-native | Native | Gaming | Yes |
| sentry-php | PHP | Backend | Yes |
| sentry-powershell | PowerShell | Backend | No |
| sentry-python | Python | Backend | Yes |
| sentry-react-native | React Native | JavaScript, Mobile | Yes |
| sentry-ruby | Ruby | Backend | Yes |
| sentry-rust | Rust | Backend | Yes |
| sentry-symfony | Symfony | Backend | Yes |
| sentry-unity | Unity | Gaming | Yes |
| sentry-unreal | Unreal | Gaming | Yes |

## Instructions

### Step 1: Gather Context

Ask the user for the following (collect all before proceeding):

**Required:**
- **Spec URL** — the specification or RFC describing the feature

**Optional:**
- **Linear initiative ID or URL** — to check which SDK teams are already tracked
- **Reference implementation PRs** — URLs to existing PRs in Python, JS, Go, or other SDKs that already implement the feature
- **Target SDKs** — if the user already knows which SDKs to target (skip the selection in Step 5)

### Step 2: Fetch and Summarize the Spec

Use `WebFetch` to read the spec URL. Present a concise summary to the user:
- Feature name
- Key requirements (bulleted list)
- Any SDK-specific considerations mentioned in the spec
- Breaking changes or deprecations

Ask the user to confirm the summary is accurate before proceeding.

### Step 3: Check Linear Initiative (if provided)

If a Linear initiative ID was provided, use the Linear MCP server to query existing projects:

```
Query: "Get initiative [identifier] with its name, description, and associated projects including their team names"
```

Record which SDK teams already have Linear projects for this initiative.

### Step 4: Check Existing GitHub Issues and PRs

For each enabled SDK repo, search GitHub for existing issues and PRs related to the feature. Use `Bash` to run:

```bash
# Search for issues
gh search issues "<feature-name>" --repo getsentry/<repo-name> --limit 5

# Search for PRs
gh search prs "<feature-name>" --repo getsentry/<repo-name> --limit 5
```

Use a short, distinctive keyword from the feature name as the search term.

### Step 5: Present Status Matrix

Show the user a table summarizing the current state:

```
| SDK Repo | Linear Project | GH Issue | GH PR | Status |
|----------|---------------|----------|-------|--------|
| sentry-python | Yes (link) | #123 | #456 (merged) | Done |
| sentry-rust | Yes (link) | None | None | Needs work |
| sentry-go | None | None | None | Needs work |
| ... | ... | ... | ... | ... |
```

Ask the user which SDKs to proceed with. Offer options:
- All that need work
- By category (Backend, Mobile, JavaScript, Gaming)
- Individual selection

### Step 6: Create GitHub Issues

For SDKs that don't have a GitHub issue yet:

1. **Draft an issue template** based on the spec summary. The template should include:
   - **Title**: Clear, concise feature name (e.g., "Implement strict trace continuation")
   - **Body**: Summary of requirements from the spec, link to the spec, acceptance criteria
   - Do NOT include reference implementation links in the issue body (those are for implementers, not the issue)

2. **Show the template to the user** and ask for confirmation or edits.

3. **Create issues** using `Bash`:
   ```bash
   gh issue create --repo getsentry/<repo-name> --title "<title>" --body "<body>"
   ```

4. Record the created issue URLs.

### Step 7: Spawn Implementation Agents

For each selected SDK, spawn a `Task` agent with `isolation: "worktree"` to implement the feature:

**Agent prompt template:**

```
You are implementing a feature in the Sentry SDK repository: getsentry/<repo-name>.

## Feature Spec
<paste spec summary>

## Reference Implementations
<paste reference PR URLs — the agent should use `gh pr diff <url>` to read them>

## Steps

1. Clone the repo if not already present:
   - Check if ~/Projects/<repo-name> exists
   - If not: `gh repo clone getsentry/<repo-name> ~/Projects/<repo-name>`
   - cd ~/Projects/<repo-name>

2. Create a feature branch: `git checkout -b <feature-branch-name>`

3. Read the reference implementations to understand the approach:
   - Use `gh pr diff <pr-url>` for each reference PR
   - Note the patterns, file locations, and test structure

4. Read the repo's CONTRIBUTING.md or CLAUDE.md for conventions.

5. Implement the feature following the repo's conventions:
   - Match the style of existing code
   - Add tests following the repo's test patterns
   - Update any relevant documentation

6. Create a draft PR:
   - Use the repo's PR template if one exists (check .github/PULL_REQUEST_TEMPLATE.md)
   - Reference the GitHub issue: "Closes getsentry/<repo-name>#<issue-number>"
   - `gh pr create --draft --title "<title>" --body "<body>"`

7. Check CI status:
   - `gh pr checks <pr-number> --repo getsentry/<repo-name> --watch`
   - If CI fails, attempt to fix and push again (up to 2 retries)

8. Report back with:
   - PR URL
   - CI status (passing/failing)
   - Any issues or notes for the user
```

Spawn agents **in parallel** using multiple `Task` tool calls. Use `model: "opus"` for each agent.

**Important**: Ask the user before spawning agents. Show them how many agents will be spawned and for which SDKs.

### Step 8: Collect Results and Link Everything

After all agents complete:

1. **Present a results table**:
   ```
   | SDK Repo | GH Issue | Draft PR | CI Status | Notes |
   |----------|----------|----------|-----------|-------|
   | sentry-rust | #123 | #456 | Passing | Ready for review |
   | sentry-go | #124 | #457 | Failing | Test timeout, needs manual fix |
   ```

2. **Link GH issues to Linear** (if Linear initiative was provided):
   - For each SDK that has both a Linear project and a GH issue, update the Linear project description to include the GH issue link

3. **Summary**: Provide a final summary of what was done and what needs manual follow-up.

## Notes

- Always confirm with the user before creating issues or PRs — never auto-create without approval
- The `gh` CLI must be authenticated (`gh auth status` to verify)
- For large rollouts (10+ SDKs), consider batching in groups of 5 to avoid rate limits
- If a reference implementation is not available, the agent should still attempt implementation based on the spec alone, but flag it for extra review
- Each implementation agent runs in an isolated worktree to avoid conflicts

## Example Usage

User: "Roll out strict trace continuation to all Backend SDKs"

Response flow:
1. Ask for spec URL and any reference PRs
2. Fetch spec, summarize requirements
3. Check GitHub for existing issues/PRs across Backend SDKs
4. Show status matrix — Python already has a merged PR, Rust has nothing, etc.
5. User selects Rust, Go, Java, Ruby, PHP, .NET, Elixir
6. Draft and confirm issue template
7. Create GH issues for each
8. Spawn implementation agents in parallel
9. Collect results, present summary table
