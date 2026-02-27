---
name: sdk-feature-rollout
description: Roll out an SDK feature across multiple Sentry SDK repositories. Use when implementing a spec across SDKs, creating GitHub issues for SDK repos, or spawning parallel implementation agents. Triggers on "rollout", "implement across SDKs", "SDK feature rollout", "cross-SDK implementation".
allowed-tools: WebFetch Task AskUserQuestion mcp__github__search_issues mcp__github__search_pull_requests mcp__github__issue_write mcp__github__issue_read mcp__github__pull_request_read mcp__github__list_issues mcp__github__list_pull_requests mcp__github__create_pull_request mcp__github__add_issue_comment mcp__linear-server__query_data mcp__linear-server__get_project mcp__linear-server__update_project
compatibility: Requires the GitHub MCP server (github/github-mcp-server) with issues and pull_requests toolsets enabled. Optionally requires the Linear MCP server for initiative tracking.
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

For each enabled SDK repo, search GitHub for existing issues and PRs related to the feature:

- Use `mcp__github__search_issues` with query `<feature-keyword> repo:getsentry/<repo-name>` to find existing issues
- Use `mcp__github__search_pull_requests` with query `<feature-keyword> repo:getsentry/<repo-name>` to find existing PRs

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

3. **Create issues** using `mcp__github__issue_write` for each repo (`getsentry/<repo-name>`).

4. Record the created issue URLs.

### Step 7: Spawn Implementation Agents

For each selected SDK, spawn a `Task` agent with `isolation: "worktree"` to implement the feature:

**Agent prompt template:**

```
You are implementing a feature in the Sentry SDK repository: getsentry/<repo-name>.

## Feature Spec
<paste spec summary>

## Reference Implementations
<paste reference PR URLs — the agent should use the GitHub MCP server to read PR diffs>

## Steps

1. Clone the repo:
   - Clone getsentry/<repo-name> into the worktree's working directory

2. Create a feature branch: git checkout -b <feature-branch-name>

3. Read the reference implementations to understand the approach:
   - Use the GitHub MCP server pull_request_read to read each reference PR diff
   - Note the patterns, file locations, and test structure

4. Read the repo's CONTRIBUTING.md or CLAUDE.md for conventions.

5. Implement the feature following the repo's conventions:
   - Match the style of existing code
   - Add tests following the repo's test patterns
   - Update any relevant documentation

6. Create a draft PR:
   - Use the repo's PR template if one exists (check .github/PULL_REQUEST_TEMPLATE.md)
   - Reference the GitHub issue: "Closes getsentry/<repo-name>#<issue-number>"
   - Use the GitHub MCP server create_pull_request with draft=true

7. Report back with:
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
   - For each SDK that has both a Linear project and a GH issue, use `mcp__linear-server__update_project` to add the GH issue link to the project description

3. **Summary**: Provide a final summary of what was done and what needs manual follow-up.

## Notes

- Always confirm with the user before creating issues or PRs — never auto-create without approval
- The GitHub MCP server must be configured and authenticated
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
