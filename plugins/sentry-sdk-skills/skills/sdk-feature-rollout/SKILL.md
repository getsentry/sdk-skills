---
name: sdk-feature-rollout
description: Roll out an SDK feature across multiple Sentry SDK repositories. Use when implementing a spec across SDKs, creating GitHub issues for SDK repos, or spawning parallel implementation agents. Triggers on "rollout", "implement across SDKs", "SDK feature rollout", "cross-SDK implementation".
allowed-tools: Read Grep Glob WebFetch Task AskUserQuestion mcp__github__search_issues mcp__github__search_pull_requests mcp__github__issue_write mcp__github__issue_read mcp__github__pull_request_read mcp__github__list_issues mcp__github__list_pull_requests mcp__github__create_pull_request mcp__github__add_issue_comment mcp__github__get_file_contents mcp__github__create_branch mcp__github__push_files mcp__github__create_or_update_file mcp__linear-server__query_data mcp__linear-server__get_project mcp__linear-server__update_project
compatibility: Requires the GitHub MCP server (github/github-mcp-server) with issues, pull_requests, and repos toolsets enabled. Optionally requires the Linear MCP server for initiative tracking.
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

## Repo Access

All operations go through the GitHub MCP server — no shell access (`Bash`) is used. Only operate on repos listed in the SDK Repos table above under the `getsentry` org.

Available GitHub MCP tools:
- **Read**: `mcp__github__get_file_contents` — read files and list directories from repos
- **Search**: `mcp__github__search_issues`, `mcp__github__search_pull_requests` — find existing issues/PRs
- **Issues**: `mcp__github__issue_write`, `mcp__github__issue_read`, `mcp__github__list_issues`, `mcp__github__add_issue_comment`
- **PRs**: `mcp__github__pull_request_read`, `mcp__github__list_pull_requests`, `mcp__github__create_pull_request`
- **Branches & Commits**: `mcp__github__create_branch`, `mcp__github__push_files`, `mcp__github__create_or_update_file`
- **Linear** (optional): `mcp__linear-server__query_data` (read-only), `mcp__linear-server__get_project`, `mcp__linear-server__update_project` (write)

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

For each enabled SDK repo, search GitHub for existing issues and PRs related to the feature using the GitHub MCP server:

- Use `mcp__github__search_issues` with query `<feature-keyword> repo:getsentry/<repo-name>` to find existing issues
- Use `mcp__github__search_pull_requests` with query `<feature-keyword> repo:getsentry/<repo-name>` to find existing PRs
- Use `mcp__github__pull_request_read` to check the state of any existing PRs (open, draft, merged, conflicting)

Use a short, distinctive keyword from the feature name as the search term. Search with multiple variants (e.g., both "strict trace continuation" and "strictTraceContinuation") to avoid missing results.

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

For each selected SDK, spawn a `Task` agent to implement the feature. All operations go through the GitHub MCP server — no local cloning or shell access.

**Do NOT use `isolation: "worktree"`** — not needed since agents work entirely via the GitHub API.

**Agent prompt template:**

```
You are implementing a feature in the Sentry SDK repository: getsentry/<repo-name>.

## Tool Restrictions

Do NOT use `Bash` or any shell commands. All operations must go through the GitHub MCP server and file tools:
- `mcp__github__get_file_contents` — read files and list directories from the repo
- `mcp__github__pull_request_read` — read reference PR diffs and details
- `mcp__github__create_branch` — create a feature branch
- `mcp__github__push_files` — push all file changes in a single commit
- `mcp__github__create_or_update_file` — create or update individual files
- `mcp__github__create_pull_request` — create the draft PR

## Feature Spec
<paste spec summary>

## Reference Implementations
<paste reference PR URLs>

## Steps

1. Read the reference implementations to understand the approach:
   - Use `mcp__github__pull_request_read` to read each reference PR diff
   - Note the patterns, file locations, and test structure

2. Explore the repo's structure and conventions:
   - Use `mcp__github__get_file_contents` to list the repo's top-level directory
   - Read CONTRIBUTING.md or CLAUDE.md for conventions
   - Explore the source tree and test directories to understand patterns

3. Create a feature branch:
   - Use `mcp__github__create_branch` on `getsentry/<repo-name>` with branch name `feat/<feature-name>`

4. Implement the feature:
   - Read existing source files with `mcp__github__get_file_contents` to understand the codebase
   - Prepare all file changes (new files and modifications)
   - Use `mcp__github__push_files` to push all changes in a single commit
   - Match the style of existing code
   - Add tests following the repo's test patterns
   - Update any relevant documentation

5. Create a draft PR:
   - Read the repo's PR template if one exists (check `.github/PULL_REQUEST_TEMPLATE.md`)
   - Use `mcp__github__create_pull_request` with `draft: true`
   - Reference the GitHub issue in the body: "Closes getsentry/<repo-name>#<issue-number>"

6. Report back with:
   - PR URL
   - Summary of changes made (files modified/created)
   - Any notes for review
```

Spawn agents **in parallel** using multiple `Task` tool calls. Use `model: "opus"` for each agent.

**Important**: Ask the user before spawning agents. Show them how many agents will be spawned and for which SDKs.

### Step 8: Wait for CI

After agents create draft PRs, check CI status for each PR using `mcp__github__pull_request_read` to see check statuses. Since there is no local test running, CI is the only verification.

If CI fails:
1. Use `mcp__github__pull_request_read` to identify failing checks
2. Spawn a `Task` agent (or resume the original) to investigate and fix. Common failure causes:
   - Existing tests that assert on baggage/header content and need updating for new fields
   - Linting or formatting issues
   - Missing imports or type errors
3. The agent reads the failing files with `mcp__github__get_file_contents`, then pushes fixes with `mcp__github__push_files`
4. Re-check CI status after the fix

Run CI-monitoring agents in the background so multiple PRs can be checked in parallel.

### Step 9: Collect Results and Link Everything

After all agents complete and CI passes:

1. **Present a results table**:
   ```
   | SDK Repo | GH Issue | Draft PR | CI Status | Notes |
   |----------|----------|----------|-----------|-------|
   | sentry-rust | #123 | #456 | Passing | Ready for review |
   | sentry-go | #124 | #457 | Failing | Test timeout, needs manual fix |
   ```

2. **Link GH issues to Linear** (if Linear initiative was provided):
   - For each SDK that has both a Linear project and a GH issue, use `mcp__linear-server__update_project` to add the GH issue link to the project description
   - Note: `mcp__linear-server__query_data` is **read-only** — always use `mcp__linear-server__update_project` for write operations

3. **Summary**: Provide a final summary of what was done and what needs manual follow-up.

## Notes

- Always confirm with the user before creating issues or PRs — never auto-create without approval
- **No shell access** — neither the skill nor spawned agents use `Bash`. All GitHub operations go through the MCP server. Do not use `git` or `gh` CLI commands
- The GitHub MCP server must be configured and authenticated with the `repos`, `issues`, and `pull_requests` toolsets enabled
- Only operate on repos listed in the SDK Repos table — do not access other repositories
- For large rollouts (10+ SDKs), consider batching in groups of 5 to avoid rate limits
- If a reference implementation is not available, the agent should still attempt implementation based on the spec alone, but flag it for extra review
- Do NOT use `isolation: "worktree"` — not needed since agents work entirely via the GitHub API
- Since there is no local test running, CI is the sole verification — always wait for CI to pass before presenting final results
- Use `mcp__github__push_files` to push all file changes in a single commit rather than `create_or_update_file` per file (which creates one commit per file)
- `mcp__linear-server__query_data` is **read-only** — always use `mcp__linear-server__update_project` for write operations

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
9. Wait for CI to pass on all draft PRs, fix any failures
10. Collect results, present summary table
