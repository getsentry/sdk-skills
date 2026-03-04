---
name: sdk-feature-implementation
description: Implement a feature across Sentry SDK repositories by spawning parallel agents. Use when you have GitHub issues or Linear projects/issues and want to create draft PRs. Triggers on "implement across SDKs", "spawn SDK agents", "SDK implementation", "parallel SDK implementation", "continue SDK rollout".
allowed-tools: Read Grep Glob Bash WebFetch Task AskUserQuestion mcp__github__search_issues mcp__github__search_pull_requests mcp__github__issue_read mcp__github__list_issues mcp__github__list_pull_requests mcp__github__pull_request_read mcp__github__create_pull_request mcp__github__add_issue_comment mcp__github__get_file_contents mcp__github__create_branch mcp__github__push_files mcp__linear-server__query_data mcp__linear-server__get_project mcp__linear-server__list_issues mcp__linear-server__save_project
compatibility: "Requires the GitHub MCP server (github/github-mcp-server) with issues, pull_requests, and repos toolsets enabled. Requires gh CLI installed and authenticated for CI log access. Recommended: the Linear MCP server (github.com/linear/linear-mcp) for tracking."
---

# SDK Feature Implementation

Implement a feature across Sentry SDK repositories by spawning parallel agents that create draft PRs.

## SDK Repos

See [SDK_REPOS.md](../../SDK_REPOS.md) for the full list of SDK repos, suffixes, categories, and enabled status. Only repos with Enabled=Yes are included by default. All repos are under the `getsentry` GitHub org.

## Repo Access

Only operate on repos listed in [SDK_REPOS.md](../../SDK_REPOS.md) under the `getsentry` org.

When performing GitHub or shell operations, read `${CLAUDE_SKILL_ROOT}/references/tools.md` for the full tools reference (GitHub MCP tools, shell access scope, allowed/disallowed commands).

## Instructions

### Step 1: Gather Context

The user needs to provide enough context to know **what** to implement and **where**. Accept any of these starting points:

- **GitHub issue URLs** (ideal) — knows exactly what to implement and where
- **Linear initiative** — fetch with `mcp__linear-server__query_data` to extract the spec URL, reference PRs, and which SDK teams have projects/issues
- **Linear project or issue** — extract spec, reference PRs, and target repo from the description
- **Spec URL + target repos** — user provides directly
- **Single repo + description** — implement in just one SDK

From the provided context, extract:
- **Spec URL** — look for links to specs/RFCs in Linear descriptions or ask the user
- **Reference implementation PRs** — look for links to existing PRs that serve as a reference
- **Target repos** — determine which SDK repos to implement in, with optional GitHub issue numbers

If starting from a Linear initiative, use `mcp__linear-server__query_data` to get the initiative details, then `mcp__linear-server__list_issues` or `mcp__linear-server__get_project` to find individual SDK team issues/projects. Cross-reference with GitHub to find matching issues using `mcp__github__search_issues`.

### Step 2: Fetch and Summarize Spec

Use `WebFetch` to read the spec URL. Present a concise summary:
- Feature name
- Key requirements (bulleted list)
- SDK-specific considerations
- Breaking changes or deprecations

Ask the user to confirm the summary is accurate before proceeding. Skip if the user already provided a sufficient description.

### Step 3: Check Implementation State

For each target repo, check for existing PRs:

- Use `mcp__github__search_pull_requests` with query `<feature-keyword> repo:getsentry/<repo-name>`
- Use `mcp__github__pull_request_read` to check status of any existing PRs

Present a matrix:

```
| SDK Repo | GH Issue | Existing PR | CI Status | Action |
|----------|----------|-------------|-----------|--------|
| sentry-rust | #123 | None | — | Implement |
| sentry-go | #124 | #200 (draft) | Failing | Fix CI |
| sentry-python | #100 | #150 (merged) | — | Skip |
```

This handles "continue from where you left off" naturally. Ask the user to confirm the action plan.

### Step 4: Spawn Implementation Agents

1. Read `${CLAUDE_SKILL_ROOT}/references/agent-prompt.md` for the implementation agent prompt template.
2. Fill in `<repo-name>`, `<spec-summary>`, `<reference-prs>`, `<feature-name>`, and `<issue-number>` for each SDK.
3. **Ask the user before spawning agents.** Show them how many agents will be spawned and for which SDKs.
4. Spawn agents **in parallel** using multiple `Task` tool calls with `model: "opus"` and `isolation: "worktree"` so each agent gets an isolated working directory.

### Step 5: Verify with CI

Local tests should have already passed during implementation. Use CI as final verification.

After agents create draft PRs, check CI status with `mcp__github__pull_request_read`.

If CI fails:

1. **Fetch the actual failure logs** — don't guess:
   ```bash
   gh run list --repo getsentry/<repo-name> --branch feat/<feature-name> --limit 5
   gh run view <run-id> --repo getsentry/<repo-name> --log-failed
   ```

2. **Spawn a fix agent** with the CI error output included in the prompt.

3. The fix agent should:
   - Read the failing files with `mcp__github__get_file_contents`
   - Optionally clone the branch locally to reproduce and verify the fix
   - Push fixes with `mcp__github__push_files`

4. Re-check CI status after the fix.

Run CI-monitoring agents in the background so multiple PRs can be checked in parallel.

### Step 6: Collect Results

Once all agents complete and CI passes:

1. **Present a results table**:
   ```
   | SDK Repo | GH Issue | Draft PR | CI Status | Notes |
   |----------|----------|----------|-----------|-------|
   | sentry-rust | #123 | #456 | Passing | Ready for review |
   | sentry-go | #124 | #457 | Failing | Test timeout, needs manual fix |
   ```

2. **Link to Linear** (if available):
   - For each SDK that has a Linear project, use `mcp__linear-server__save_project` to add the PR links to the project description
   - Note: `mcp__linear-server__query_data` is **read-only** — always use `mcp__linear-server__save_project` for writes

3. **Summary**: What was done and what needs manual follow-up.

## Notes

- Always confirm with the user before spawning agents — never auto-create without approval
- **Use GitHub MCP for all GitHub operations** (issues, PRs, search, branches, pushing code). `Bash` is only for local testing, linting, and CI log access (`gh run view` / `gh api`)
- Only operate on repos listed in [SDK_REPOS.md](../../SDK_REPOS.md)
- For large rollouts (10+ SDKs), consider batching in groups of 5 to avoid rate limits
- If a reference implementation is not available, the agent should still attempt implementation based on the spec alone, but flag it for extra review
- Implementation agents use `isolation: "worktree"` for isolated working directories
- Run tests locally before creating PRs. CI serves as final verification
- Use `mcp__github__push_files` to push all file changes in a single commit
- Use the **`linear-sdk-rollout`** skill first if Linear projects/issues don't exist yet

## Example Usage

User: "Implement strict trace continuation across Rust, Go, and Java SDKs"

Response flow:
1. Gather context — ask for GitHub issue URLs, or find them via Linear initiative
2. Fetch spec, confirm summary with user
3. Check existing PRs — Rust has nothing, Go has a failing draft, Java has nothing
4. Confirm plan: spawn 2 new agents (Rust, Java) + 1 fix agent (Go)
5. Spawn agents in parallel with worktree isolation
6. Monitor CI, spawn fix agents for failures
7. Present results table, link to Linear
