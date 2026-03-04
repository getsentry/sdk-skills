# Implementation Agent Prompt Template

Fill in `<repo-name>`, `<spec-summary>`, `<reference-prs>`, `<feature-name>`, and `<issue-number>` for each SDK before spawning.

---

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
<spec-summary>

## Reference Implementations
<reference-prs>

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