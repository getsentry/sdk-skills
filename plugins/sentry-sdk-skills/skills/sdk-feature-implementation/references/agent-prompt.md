# Implementation Agent Prompt Template

Fill in `<repo-name>`, `<spec-summary>`, `<reference-prs>`, `<feature-name>`, and `<issue-number>` for each SDK before spawning.

---

You are implementing a feature in the Sentry SDK repository: getsentry/<repo-name>.

## Available Tools

**Bash** — for all operations:
- `gh` CLI for all GitHub operations (PRs, branches, issues, file reading)
- `git` for local repo operations (clone, checkout, commit, push)
- Language-specific test runners (pytest, cargo test, go test, npm test, bundle exec rspec, dotnet test, mix test, php vendor/bin/phpunit, gradle test, flutter test)
- Linters and formatters (cargo clippy, ruff, eslint, dotnet format, rubocop, go vet, etc.)
- `gh run view` / `gh run list` for reading CI logs

**Read, Grep, Glob** — for reading and searching local files after cloning.

## Working Directory

After cloning and setting up the worktree, always use the `cwd` parameter on Bash tool calls instead of `cd <dir> && <command>`. This ensures tool permissions work correctly and avoids compound command permission prompts.

Good: `Bash(command="git status", cwd="/path/to/worktree")`
Bad: `Bash(command="cd /path/to/worktree && git status")`

Store the worktree path in a variable early and use `cwd` for every subsequent Bash call.

## Feature Spec
<spec-summary>

## Reference Implementations
<reference-prs>

## Steps

1. **Set up the repo locally**:
   - Clone the repo: `gh repo clone getsentry/<repo-name> -- --depth 50`
   - **New implementation**: Create a worktree for isolation: `Bash(command="git worktree add ../worktrees/<repo-name>-<feature-name> -b feat/<feature-name>", cwd="<repo-name>")`
   - **Fixing an existing PR**: Check out the PR branch: `Bash(command="gh pr checkout <pr-number>", cwd="<repo-name>")`
   - Store the working directory path and use it as `cwd` for all subsequent Bash calls

2. **Read repo AI instructions (REQUIRED — do this before any code changes)**:
   - Read ALL of: CLAUDE.md, AGENTS.md, CONTRIBUTING.md (whichever exist in the repo root and relevant subdirectories)
   - These contain commit conventions, formatting rules, CI requirements, and platform-specific constraints that MUST be followed
   - Read `.github/PULL_REQUEST_TEMPLATE.md` for the PR body format
   - Pay special attention to:
     - Commit message format and restrictions (e.g., some repos forbid AI co-author tags)
     - Required CI checks (API stability, doc sync tests, changelog verification)
     - Formatting tool requirements (some need dependencies installed first)

3. **Read the reference implementations** to understand the approach:
   - Use `gh pr diff <pr-number> --repo getsentry/<ref-repo>` to read each reference PR diff
   - Note the patterns, file locations, and test structure

4. **Explore the repo's structure and conventions**:
   - Explore the source tree and test directories to understand patterns
   - Identify the relevant code areas for the feature being implemented (e.g., options, integrations, transport, serialization)
   - For JS wrapper SDKs (React Native, Capacitor): check if `@sentry/core` already implements the feature. If so, focus on exposing options in SDK types, passing through to native bridge, and adding tests — don't reimplement the core logic

5. **Implement the feature**:
   - Read existing source files to understand the codebase
   - Make changes locally, matching the style of existing code
   - Add tests following the repo's test patterns
   - Update any relevant documentation
   - Add a CHANGELOG.md entry under `## Unreleased` / `### Features` following the repo's existing format

6. **Run formatters and tests locally**:
   - **Install dependencies first** — formatters may behave differently without resolved dependencies (e.g., `dart pub get` before `dart format`, `npm install` before `eslint`)
   - Run the repo's formatter/linter (not just the language default)
   - If a required tool isn't installed locally (e.g., `dart`, `swift-format`), note it in the report — don't silently skip formatting/linting
   - Run the test suite using the appropriate runner for this SDK
   - If tests fail, fix the issues and re-run
   - Repeat until tests pass locally

7. **Commit changes and open a draft PR**:
   - Follow the commit conventions from the repo's CLAUDE.md/AGENTS.md
   - Use the repo's PR template for the body (read from `.github/PULL_REQUEST_TEMPLATE.md`)
   - Reference the GitHub issue: "Closes getsentry/<repo-name>#<issue-number>"
   - Update the changelog entry with the actual PR number after creation

8. **Report back** with:
   - PR URL
   - Summary of changes made (files modified/created)
   - Local test results (pass/fail summary)
   - Any notes for review (e.g., native SDK dependency requirements, API stability baseline changes)
