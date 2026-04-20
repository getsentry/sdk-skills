---
name: sdk-feature-implementation
description: Implement a feature across Sentry SDK repositories by spawning parallel agents. Use when you have Linear initiatives/projects/issues and want to create draft PRs. Triggers on "implement across SDKs", "spawn SDK agents", "SDK implementation", "parallel SDK implementation", "continue SDK rollout".
allowed-tools: Read Grep Glob Bash WebFetch Task AskUserQuestion mcp__linear-server__query_data mcp__linear-server__get_project mcp__linear-server__get_issue mcp__linear-server__list_issues mcp__linear-server__get_initiative mcp__linear-server__list_comments mcp__linear-server__save_project mcp__linear-server__update_project
compatibility: "Requires the Linear MCP server (github.com/linear/linear-mcp) for context gathering and tracking. Requires gh CLI installed and authenticated for all GitHub operations (PRs, branches, CI logs)."
---

# SDK Feature Implementation

Implement a feature across Sentry SDK repositories by spawning parallel agents that create draft PRs.

## Repo Access

Only operate on repos listed in [SDK_REPOS.md](../../SDK_REPOS.md). Only repos with Enabled=Yes are included by default. All repos are under the `getsentry` GitHub org.

Use `gh` CLI for all GitHub operations (PRs, branches, CI logs). Use Linear MCP for all context gathering and tracking.

## Instructions

### Step 1: Gather Context

**Linear MCP is the primary source for all context gathering.** The user needs to provide enough context to know **what** to implement and **where**. Accept any of these starting points:

- **Linear initiative** (ideal) — fetch with `mcp__linear-server__get_initiative` to get the overview, then drill into projects/issues per SDK team
- **Linear project or issue** — extract spec, reference PRs, and target repo from the description
- **Spec URL + target repos** — user provides directly
- **Single repo + description** — implement in just one SDK

From the provided context, extract:
- **Spec URL** — look for links to specs/RFCs in Linear issue/project descriptions
- **Reference implementation PRs** — look for PR links in Linear issue attachments or descriptions
- **Target repos** — determine which SDK repos to implement in
- **GitHub issue numbers** — found in Linear issue attachments (GitHub links are auto-attached)

**Context gathering flow:**
1. `mcp__linear-server__get_initiative` — get initiative overview and linked projects
2. `mcp__linear-server__get_project` — get project details for target SDK teams
3. `mcp__linear-server__get_issue` — get detailed requirements, spec links, and GitHub issue attachments from each issue
4. `mcp__linear-server__list_comments` — check for additional context or decisions in issue comments

Linear issues have GitHub issue/PR links as **attachments** — use these to find the corresponding GitHub issue numbers without needing to search GitHub directly.

### Step 2: Fetch and Summarize Spec

If a spec URL was found, use `WebFetch` to read it. Otherwise, gather requirements from Linear issue descriptions and `gh issue view <number> --repo getsentry/<repo-name>` for GitHub issue details. Present a concise summary:
- Feature name
- Key requirements (bulleted list)
- SDK-specific considerations
- Breaking changes or deprecations

Ask the user to confirm the summary is accurate before proceeding.

### Step 2.5: Identify SDK Dependencies

Hybrid SDKs depend on native SDKs for their platform layers:
- **sentry-react-native** and **sentry-capacitor** wrap `@sentry/core` (JS) + sentry-cocoa (iOS) + sentry-java (Android)
- **sentry-dart** (Flutter) has its own Dart implementation but uses sentry-cocoa (iOS) and sentry-java (Android) for native crash reporting and trace context in crash envelopes
- **sentry-unity** and **sentry-kotlin-multiplatform** also depend on sentry-cocoa and sentry-java

When implementing across both hybrid and native SDKs:
- **Implement native SDKs (Cocoa, Java) first** — hybrid SDKs can't compile native bridge code until those are released
- **JS wrapper SDKs** (React Native, Capacitor) may already have the feature via `@sentry/core` — check before implementing from scratch. If `@sentry/core` already has it, the work is: expose options in SDK types, pass through to native bridge, add tests
- **Hybrid SDK PRs should note** in "Next steps" which native SDK PRs must merge and be released before the hybrid SDK is fully functional
- **Only spawn ONE agent per repo.** If a repo already has a worktree/branch from a previous run, reuse it

### Step 3: Check Implementation State

For each target repo, check for existing PRs:

- **First check Linear** — issue attachments contain links to GitHub PRs that were auto-linked
- **Then check GitHub** for PR status using `gh pr list --repo getsentry/<repo-name> --search "<feature-keyword>" --json number,title,state,isDraft,url`
- For PR details: `gh pr view <number> --repo getsentry/<repo-name> --json state,isDraft,mergeable,statusCheckRollup`

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

1. Read `references/agent-prompt.md` for the implementation agent prompt template.
2. Fill in `<repo-name>`, `<spec-summary>`, `<reference-prs>`, `<feature-name>`, and `<issue-number>` for each SDK.
3. **Ask the user before spawning agents.** Show them how many agents will be spawned and for which SDKs.
4. Spawn agents **in parallel**. Each agent clones the repo and creates a git worktree internally for isolation.

### Step 5: Verify with CI

Local tests should have already passed during implementation. Use CI as final verification.

After agents create draft PRs, check CI status with `gh pr view <number> --repo getsentry/<repo-name> --json statusCheckRollup`.

If CI fails:

1. **Fetch the actual failure logs** — don't guess:
   ```bash
   gh run list --repo getsentry/<repo-name> --branch feat/<feature-name> --limit 5
   gh run view <run-id> --repo getsentry/<repo-name> --log-failed
   ```

2. **Fix the issue directly** or spawn a fix agent with the CI error output included in the prompt.

3. The fix should:
   - Read the failing files locally in the worktree
   - Reproduce and verify the fix locally
   - Push fixes with `git push`

4. Re-check CI status after the fix.

**Expect 2-3 CI fix iterations.** Common issues:
- **Formatting** — Run formatters with resolved dependencies (e.g., `dart pub get` before `dart format`)
- **Linting** — Unnecessary imports, public API exposure for `@internal` functions
- **API stability** — Some SDKs (Cocoa, Java) have API baseline checks; regenerate if you changed public API
- **Doc sync tests** — Some SDKs check that all options are documented in sentry-docs; add to allowlist if docs PR is separate
- **Changelog** — Some SDKs verify changelog entries exist for new features

Run CI-monitoring agents in the background so multiple PRs can be checked in parallel.

### Step 5.5: Address Review Feedback

After CI passes, check for automated review feedback (Cursor bot, Danger, Sentry bot, etc.):

```bash
gh api repos/getsentry/<repo-name>/pulls/<number>/comments
gh api repos/getsentry/<repo-name>/issues/<number>/comments
```

- Address valid feedback with code fixes and reply referencing the fix commit
- Reply to false positives with reasoning
- Common automated feedback: missing option allowlists, public API exposure, test coverage gaps

### Step 6: Create Documentation PRs

For each SDK implementation, check if sentry-docs needs updates:

1. Search sentry-docs for existing documentation related to the feature (e.g., configuration options pages, feature-specific guides, platform includes)
2. Add or update documentation as needed — new options, new guides, or updated platform-specific includes
3. Create docs PRs following the sentry-docs PR template (`.github/PULL_REQUEST_TEMPLATE.md`)
4. Link docs PRs from the SDK PRs

Create separate docs PRs per SDK — don't bundle multiple platforms into one PR.

### Step 7: Collect Results

Once all agents complete and CI passes:

1. **Present a results table**:
   ```
   | SDK Repo | GH Issue | Draft PR | CI Status | Docs PR | Notes |
   |----------|----------|----------|-----------|---------|-------|
   | sentry-rust | #123 | #456 | Passing | #789 | Ready for review |
   | sentry-go | #124 | #457 | Failing | — | Test timeout, needs manual fix |
   ```

2. **Link to Linear** (if available):
   - For each SDK that has a Linear project, use `mcp__linear-server__save_project` to add the PR links to the project description
   - Note: `mcp__linear-server__query_data` is **read-only** — always use `mcp__linear-server__save_project` for writes

3. **Summary**: What was done and what needs manual follow-up.

## Notes

- Always confirm with the user before spawning agents — never auto-create without approval
- **Use Linear MCP for all context gathering** (specs, requirements, issue details, project status). Use `gh` CLI for all GitHub operations (PRs, branches, CI logs, pushing code)
- Only operate on repos listed in [SDK_REPOS.md](../../SDK_REPOS.md)
- For large rollouts (10+ SDKs), consider batching in groups of 5 to avoid rate limits
- If a reference implementation is not available, the agent should still attempt implementation based on the spec alone, but flag it for extra review
- Implementation agents clone the repo and use `git worktree` for isolated working directories
- Run tests locally before creating PRs. CI serves as final verification
- Use the **`linear-sdk-rollout`** skill first if Linear projects/issues don't exist yet

## Example Usage

User: "Implement strict trace continuation across Rust, Go, and Java SDKs"

Response flow:
1. Gather context — get initiative, projects, and issues from Linear MCP
2. Fetch spec, confirm summary with user
3. Check existing PRs — Rust has nothing, Go has a failing draft, Java has nothing
4. Confirm plan: spawn 2 new agents (Rust, Java) + 1 fix agent (Go)
5. Spawn agents in parallel with worktree isolation
6. Monitor CI, fix failures (expect 2-3 iterations)
7. Create docs PRs for each SDK
8. Present results table, link to Linear
