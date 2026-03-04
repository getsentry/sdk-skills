# Tools Reference

## GitHub MCP Tools (Primary)

Use the GitHub MCP server for **all** GitHub operations — reading files, searching, issues, PRs, branches, and pushing code:
- **Read**: `mcp__github__get_file_contents` — read files and list directories from repos
- **Search**: `mcp__github__search_issues`, `mcp__github__search_pull_requests` — find existing issues/PRs
- **Issues**: `mcp__github__issue_write`, `mcp__github__issue_read`, `mcp__github__list_issues`, `mcp__github__add_issue_comment`
- **PRs**: `mcp__github__pull_request_read`, `mcp__github__list_pull_requests`, `mcp__github__create_pull_request`
- **Branches & Commits**: `mcp__github__create_branch`, `mcp__github__push_files`
- **Linear** (recommended): `mcp__linear-server__query_data` (read-only), `mcp__linear-server__get_project`, `mcp__linear-server__save_project` (write)

## Shell Access (Scoped)

`Bash` is allowed **only** for CI log access and local testing. Do not use `gh` or `git` for operations that the GitHub MCP server can handle.

### CI log access

Use `gh` CLI only for reading CI output (not available via MCP):
```bash
# List recent workflow runs for a branch
gh run list --repo getsentry/<repo-name> --branch <branch> --limit 5

# View logs for failed steps only
gh run view <run-id> --repo getsentry/<repo-name> --log-failed

# Fetch check run annotations (specific failure lines)
gh api repos/getsentry/<repo-name>/check-runs/<check-run-id>/annotations
```

### Local testing

Clone and run tests before pushing:
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

### Linters and formatters
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

### Not allowed via Bash
- GitHub operations (issues, PRs, search, pushing code) — use MCP instead
- Arbitrary shell commands unrelated to testing, linting, or CI logs
- Network access beyond `git clone` and `gh run`/`gh api`
- Installing system-level packages