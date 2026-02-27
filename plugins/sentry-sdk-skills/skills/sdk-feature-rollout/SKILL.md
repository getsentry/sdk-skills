---
name: sdk-feature-rollout
description: Roll out an SDK feature across multiple Sentry SDK repositories. Use when implementing a spec across SDKs, creating GitHub issues for SDK repos, or spawning parallel implementation agents. Triggers on "rollout", "implement across SDKs", "SDK feature rollout", "cross-SDK implementation".
allowed-tools: Read Grep Glob Bash WebFetch Task AskUserQuestion mcp__github__search_issues mcp__github__search_pull_requests mcp__github__issue_write mcp__github__issue_read mcp__github__pull_request_read mcp__github__list_issues mcp__github__list_pull_requests mcp__github__create_pull_request mcp__github__add_issue_comment mcp__github__get_file_contents mcp__github__create_branch mcp__github__push_files mcp__linear-server__query_data mcp__linear-server__get_project mcp__linear-server__update_project
compatibility: Requires the GitHub MCP server (github/github-mcp-server) with issues, pull_requests, and repos toolsets enabled. Requires `gh` CLI installed and authenticated for CI log access. Optionally requires the Linear MCP server for initiative tracking.
---

# SDK Feature Rollout

Automates rolling out a feature from spec to implementation across multiple Sentry SDK repositories.

## SDK Repos

See [SDK_REPOS.md](../../SDK_REPOS.md) for the full list of SDK repos, suffixes, categories, and enabled status. Only repos with Enabled=Yes are included by default. All repos are under the `getsentry` GitHub org.

## Repo Access

Only operate on repos listed in [SDK_REPOS.md](../../SDK_REPOS.md) under the `getsentry` org.

### GitHub MCP Tools (Primary)

Use the GitHub MCP server for **all** GitHub operations — reading files, searching, issues, PRs, branches, and pushing code:
- **Read**: `mcp__github__get_file_contents` — read files and list directories from repos
- **Search**: `mcp__github__search_issues`, `mcp__github__search_pull_requests` — find existing issues/PRs
- **Issues**: `mcp__github__issue_write`, `mcp__github__issue_read`, `mcp__github__list_issues`, `mcp__github__add_issue_comment`
- **PRs**: `mcp__github__pull_request_read`, `mcp__github__list_pull_requests`, `mcp__github__create_pull_request`
- **Branches & Commits**: `mcp__github__create_branch`, `mcp__github__push_files`
- **Linear** (optional): `mcp__linear-server__query_data` (read-only), `mcp__linear-server__get_project`, `mcp__linear-server__update_project` (write)

### Shell Access (Scoped)

`Bash` is allowed **only** for CI log access and local testing. Do not use `gh` or `git` for operations that the GitHub MCP server can handle.

**CI log access** — use `gh` CLI only for reading CI output (not available via MCP):
```bash
# List recent workflow runs for a branch
gh run list --repo getsentry/<repo-name> --branch <branch> --limit 5

# View logs for failed steps only
gh run view <run-id> --repo getsentry/<repo-name> --log-failed

# Fetch check run annotations (specific failure lines)
gh api repos/getsentry/<repo-name>/check-runs/<check-run-id>/annotations
```

**Local testing** — clone and run tests before pushing:
```bash
# Clone the repo (shallow, specific branch)
git clone --depth 1 --branch <branch> https://github.com/getsentry/<repo-name>.git

# Language-specific test runners
pytest                          # Python
cargo test                      # Rust
go test ./...                   # Go
npm test                        # JavaScript
bundle exec rspec               # Ruby
dotnet test                     # .NET
mix test                        # Elixir
php vendor/bin/phpunit          # PHP
gradle test                     # Java/Android
flutter test                    # Dart/Flutter
```

**Linters and formatters:**
```bash
cargo clippy                    # Rust
cargo fmt --check               # Rust
ruff check / ruff format        # Python
npm run lint / npm run format   # JavaScript
dotnet format                   # .NET
mix format --check-formatted    # Elixir
php-cs-fixer fix --dry-run      # PHP
rubocop                         # Ruby
go vet / gofmt                  # Go
```

**Not allowed via Bash:**
- GitHub operations (issues, PRs, search, pushing code) — use MCP instead
- Arbitrary shell commands unrelated to testing, linting, or CI logs
- Network access beyond `git clone` and `gh run`/`gh api`
- Installing system-level packages

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

For each selected SDK, spawn a `Task` agent to implement the feature. Agents use the GitHub MCP server for remote operations and scoped `Bash` for local testing.

Use `isolation: "worktree"` so each agent gets an isolated working directory for cloning and testing.

**Agent prompt template:**

```
You are implementing a feature in the Sentry SDK repository: getsentry/<repo-name>.

## Available Tools

**GitHub MCP** — for all GitHub operations (reading, searching, issues, PRs, branches, pushing code):
- `mcp__github__get_file_contents` — read files and list directories from the repo
- `mcp__github__pull_request_read` — read reference PR diffs and details
- `mcp__github__create_branch` — create a feature branch
- `mcp__github__push_files` — push all file changes in a single commit
- `mcp__github__create_pull_request` — create the draft PR

**Bash** — only for local testing, linting, and CI log access:
- `git clone --depth 1` to clone the repo locally for testing
- Language-specific test runners (pytest, cargo test, go test, npm test, bundle exec rspec, dotnet test, mix test, php vendor/bin/phpunit, gradle test, flutter test)
- Linters and formatters (cargo clippy, ruff, eslint, dotnet format, rubocop, go vet, etc.)
- `gh run view` / `gh api` for reading CI logs (only when CI fails)
- Do NOT use Bash for GitHub operations (issues, PRs, pushing) — use MCP instead

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

5. Run tests locally:
   - Clone the repo: `git clone --depth 1 --branch feat/<feature-name> https://github.com/getsentry/<repo-name>.git`
   - Install dependencies if needed (follow the repo's setup instructions)
   - Run the test suite using the appropriate runner for this SDK
   - If tests fail, fix the issues, push fixes with `mcp__github__push_files`, and re-run
   - Repeat until tests pass locally

6. Create a draft PR:
   - Read the repo's PR template if one exists (check `.github/PULL_REQUEST_TEMPLATE.md`)
   - Use `mcp__github__create_pull_request` with `draft: true`
   - Reference the GitHub issue in the body: "Closes getsentry/<repo-name>#<issue-number>"

7. Report back with:
   - PR URL
   - Summary of changes made (files modified/created)
   - Local test results (pass/fail summary)
   - Any notes for review
```

Spawn agents **in parallel** using multiple `Task` tool calls. Use `model: "opus"` and `isolation: "worktree"` for each agent.

**Important**: Ask the user before spawning agents. Show them how many agents will be spawned and for which SDKs.

### Step 8: Verify with CI

Local tests should have already passed in Step 7. CI serves as final verification in the full repo environment.

After agents create draft PRs, check CI status using `mcp__github__pull_request_read`.

If CI fails:

1. **Fetch the actual failure logs** — don't guess:
   ```bash
   # Find the failed run
   gh run list --repo getsentry/<repo-name> --branch feat/<feature-name> --limit 5

   # Get the failure output
   gh run view <run-id> --repo getsentry/<repo-name> --log-failed
   ```

2. **Spawn a fix agent** (or resume the original) with the CI error output included in the prompt. Common failure causes:
   - Existing tests that assert on baggage/header content and need updating for new fields
   - Linting or formatting issues not caught locally
   - Integration tests or CI-only test suites
   - Missing imports or type errors in untested code paths

3. The fix agent:
   - Reads the failing files with `mcp__github__get_file_contents`
   - Optionally clones the branch locally to reproduce and verify the fix
   - Pushes fixes with `mcp__github__push_files`

4. Re-check CI status after the fix. Use `gh run view` to monitor the new run.

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
- **Use GitHub MCP for all GitHub operations** (issues, PRs, search, branches, pushing code). `Bash` is only for local testing, linting, and CI log access (`gh run view` / `gh api`)
- The GitHub MCP server must be configured and authenticated with the `repos`, `issues`, and `pull_requests` toolsets enabled
- The `gh` CLI must be installed and authenticated for CI log access only
- Only operate on repos listed in [SDK_REPOS.md](../../SDK_REPOS.md) — do not access other repositories
- For large rollouts (10+ SDKs), consider batching in groups of 5 to avoid rate limits
- If a reference implementation is not available, the agent should still attempt implementation based on the spec alone, but flag it for extra review
- Implementation agents use `isolation: "worktree"` to get an isolated working directory for cloning and local testing
- Run tests locally before creating PRs. CI serves as final verification — always wait for CI to pass before presenting final results
- Use `mcp__github__push_files` to push all file changes in a single commit
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
