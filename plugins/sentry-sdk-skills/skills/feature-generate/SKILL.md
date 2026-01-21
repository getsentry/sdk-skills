---
name: sdk-feature-generate
description: Generate idiomatic implementation code for ONE Sentry SDK based on develop docs and reference implementation. Analyzes SDK patterns, follows conventions, generates tests, runs linting, and commits changes. Works standalone or with context from sdk-feature-status.
model: sonnet
allowed-tools: Read Grep Glob Bash Write Edit Task TodoWrite
compatibility: Requires gh CLI, git, and SDK-specific linting tools (black/ruff for Python, eslint for JS, etc.).
---

# SDK Feature Code Generator

Generate idiomatic implementation code for a specific Sentry SDK based on develop docs and reference implementation.

## When to Use This Skill

Use this skill when:
- You need to implement a feature in a specific SDK
- You have a develop doc and reference implementation
- You want idiomatic, linted code following SDK conventions
- You're working through SDK alignment one SDK at a time

## Planning Complex Implementations

For complex or unfamiliar features, consider using EnterPlanMode before generating code:

**Use EnterPlanMode when:**
- The feature involves significant architectural changes
- You're unfamiliar with the target SDK's codebase structure
- Multiple approaches are possible and you need to design the best one
- The feature touches many files or subsystems

**In plan mode, you can:**
- Thoroughly explore the target SDK's codebase
- Study similar existing features
- Design the implementation approach
- Identify all files that need changes
- Get user approval on the approach before writing code

**Skip plan mode when:**
- The feature is straightforward and well-defined
- You've already explored the SDK in a previous step
- The reference implementation provides a clear pattern to follow

## Shared Context System

This skill can:
- **Read** from `.sdk-align/context.json` (created by `sdk-feature-status`)
- **Write** to `.sdk-align/implementations.json` (tracks generated implementations)
- **Work standalone** if develop doc and reference are provided directly

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
- Target SDK (e.g., "python", "go", "java")
- Develop doc (PR/commit/URL) OR read from `.sdk-align/context.json`
- Reference implementation (SDK + PR) OR read from context

**Optional inputs:**
- Branch name (auto-generates if not provided: `feat/<feature-slug>`)
- Push to remote (default: false, just commit locally)

**If context file exists:**

Use the Read tool to check for and read `.sdk-align/context.json`.

Extract feature name, develop doc, and reference implementation from context.

**If no context file:**
Ask user for:
1. Develop doc URL/path
2. Reference SDK and PR URL

### Step 2: Fetch Reference Implementation

Clone or fetch the reference SDK repository:

```bash
# Clone reference SDK
gh repo clone <REFERENCE_SDK_REPO> /tmp/reference-sdk -- --depth 1

# Fetch the reference PR files
gh pr view <PR_NUM> --repo <REFERENCE_SDK_REPO> --json files --jq '.files[] | {filename, additions, deletions}'
```

Analyze reference implementation:
- **Core files changed** (implementation, config, utils)
- **Test files added/changed**
- **Code patterns** (class names, function signatures, variable names)
- **Configuration options** (new config fields, defaults)
- **Documentation comments** (docstrings, JSDoc)

### Step 3: Clone Target SDK Repository

```bash
# Clone target SDK
gh repo clone <TARGET_SDK_REPO> /tmp/<target-sdk-name> -- --depth 1
cd /tmp/<target-sdk-name>

# Create feature branch
git checkout -b <branch-name>
```

### Step 4: Analyze Target SDK Patterns

Study the target SDK to understand conventions:

#### File Structure
```bash
# List directory structure
tree . -L 3 -I "node_modules|__pycache__|.git|vendor|build|dist"

# Find similar features
find . -name "*integration*" -o -name "*config*" -o -name "*option*" | head -20
```

#### Naming Conventions
```bash
# Check for naming patterns
grep -r "class.*Integration" . | head -5
grep -r "def " . | head -20  # Python
grep -r "function " . | head -20  # JavaScript
grep -r "func " . | head -20  # Go
```

Detect:
- **Python**: `snake_case` for functions/variables, `PascalCase` for classes
- **JavaScript**: `camelCase` for functions/variables, `PascalCase` for classes
- **Go**: `PascalCase` for exported, `camelCase` for unexported
- **Ruby**: `snake_case` for methods, `PascalCase` for classes
- **PHP**: `camelCase` or `snake_case` depending on framework
- **Java/C#**: `PascalCase` for classes/methods, `camelCase` for variables

#### Test Framework
```bash
# Identify test framework
ls -la | grep -iE "(test|spec)"
cat package.json | jq '.devDependencies' 2>/dev/null  # JavaScript
cat pyproject.toml | grep -A5 "\[tool.pytest\]" 2>/dev/null  # Python
cat Gemfile | grep -i test 2>/dev/null  # Ruby
```

Common frameworks:
- **Python**: pytest
- **JavaScript**: jest, vitest, mocha
- **Ruby**: rspec
- **PHP**: phpunit
- **Go**: testing package
- **Java**: junit

#### Linting Configuration
```bash
# Find linting configs
ls -la | grep -iE "(eslint|prettier|ruff|black|rubocop|golangci)"
cat .eslintrc* 2>/dev/null
cat pyproject.toml | grep -A10 "\[tool.ruff\]" 2>/dev/null
```

### Step 5: Generate Idiomatic Implementation

Based on reference implementation + target SDK patterns, generate idiomatic code:

#### Core Implementation Files

Analyze the reference implementation and adapt it to the target SDK's conventions:

**Naming conventions:**
- Study existing files in the target SDK (use Glob/Grep to find similar features)
- Follow the SDK's naming patterns (e.g., snake_case for Python, camelCase for JavaScript)
- Match function/class naming style from similar features

**Type systems:**
- Use the SDK's type annotation approach (type hints for Python, TypeScript types for JS, etc.)
- Study existing code to understand how types are documented

**File structure:**
- Place files in locations consistent with similar features
- Match the SDK's organization (e.g., feature modules, utilities, configuration)

**Documentation:**
- Add comments/docstrings matching the SDK's style
- Study existing functions to see docstring format

#### Configuration/Options

Identify configuration options from the develop doc and reference implementation. Add them to the SDK's configuration system by:
- Finding where options are defined (use Grep to search for "options", "config", etc.)
- Adding new options following existing patterns
- Setting appropriate defaults

#### Tests

Generate comprehensive tests that mirror the reference implementation's test coverage:
- Identify the test framework used (pytest, jest, rspec, junit, etc.)
- Match test file naming conventions (e.g., `test_*.py`, `*.test.ts`, `*_spec.rb`)
- Follow assertion patterns used in existing tests
- Cover the same scenarios as the reference implementation
- Use SDK-specific test fixtures and helpers

### Step 6: Run Linting and Tests

Execute SDK-specific linting and testing:

**Python:**
```bash
# Format code
black sentry_sdk/tracing/strict_trace.py
black tests/test_strict_trace.py

# Lint
ruff check sentry_sdk/tracing/strict_trace.py
ruff check tests/test_strict_trace.py

# Type check
mypy sentry_sdk/tracing/strict_trace.py

# Run tests
pytest tests/test_strict_trace.py -v
```

**JavaScript:**
```bash
# Lint and format
npm run lint:fix
npm run prettier:fix

# Type check
npm run type-check

# Run tests
npm test -- --testPathPattern=strictTrace
```

**Go:**
```bash
# Format
gofmt -w .

# Lint
golangci-lint run

# Test
go test ./...
```

**Ruby:**
```bash
# Format and lint
bundle exec rubocop -a

# Test
bundle exec rspec spec/sentry/strict_trace_spec.rb
```

**Fix any linting/test errors** before proceeding. If tests fail, debug and fix.

### Step 7: Commit Changes

Create a properly formatted commit:

```bash
git add .
git commit -m "$(cat <<'EOF'
feat(<sdk>): Implement <feature-name>

Implements <feature-name> following the specification in develop docs.

Key changes:
- Add <key-change-1>
- Implement <key-change-2>
- Add tests for <test-scenarios>

Based on reference implementation: <reference-pr-url>
Spec: <develop-doc-url>

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

**Commit message format:**
- **Type**: `feat`, `fix`, `ref`, `test`, `docs`
- **Scope**: SDK name or component
- **Description**: Brief summary
- **Body**: Key changes, links to develop doc and reference PR
- **Co-author**: Claude attribution

### Step 8: Push Branch (Optional)

If user requested push:

```bash
git push -u origin <branch-name>
```

Otherwise, just commit locally.

### Step 9: Update Implementations Context

Write implementation details to `.sdk-align/implementations.json`:

```json
{
  "version": "1.0",
  "implementations": {
    "python": {
      "sdk": "python",
      "feature": "Strict Trace Propagation",
      "branch": "feat/strict-trace-continuation",
      "commit": "abc123def456",
      "filesChanged": [
        "sentry_sdk/tracing/strict_trace.py",
        "sentry_sdk/consts.py",
        "tests/test_strict_trace.py"
      ],
      "pushed": false,
      "generatedAt": "2026-01-18T00:15:00Z",
      "lintingPassed": true,
      "testsPassed": true
    }
  }
}
```

Update or create this file with the new implementation.

### Step 10: Output Summary

Display to user:
- ✅ Implementation generated for `<sdk>`
- Branch: `<branch-name>`
- Files changed: `<count>` files
- Commit: `<commit-sha>`
- Linting: Passed ✓
- Tests: Passed ✓ (X tests)
- Pushed: Yes/No
- Context saved to `.sdk-align/implementations.json`

Suggest next step:
```
Next: Run /sdk-feature-pr python to create a pull request
```

## Guidelines

### Progress Tracking

**Use TodoWrite to track the implementation process:**

Create a todo list with the following steps:
1. Gather required information
2. Fetch reference implementation
3. Clone target SDK repository
4. Analyze target SDK patterns
5. Generate idiomatic implementation
6. Run linting and tests
7. Commit changes
8. Push branch (if requested)
9. Update implementations context

Mark each step as in_progress when starting and completed when finished. This helps users track progress through the multi-step generation process.

### Code Generation Principles

1. **Stay Idiomatic**
   - Don't translate code literally
   - Use language-specific idioms and patterns
   - Follow SDK conventions for naming, structure, and style

2. **Match Existing Patterns**
   - Study similar features in target SDK
   - Use same file structure and organization
   - Follow existing test patterns

3. **Respect Type Systems**
   - Python: Use type hints/comments
   - TypeScript: Use proper type annotations
   - Go: Follow exported/unexported conventions
   - Java/C#: Use strong typing throughout

4. **Complete Implementation**
   - Core functionality
   - Configuration options
   - Tests (unit + integration if applicable)
   - Documentation comments
   - Error handling

5. **SDK Philosophy**
   - Some SDKs are explicit (Python, Go)
   - Some are "magical" (Ruby, PHP frameworks)
   - Adapt implementation to SDK's design philosophy

### Handling Edge Cases

**Reference implementation not accessible:**
- Use only develop doc to generate code
- Ask user for implementation guidance
- Generate scaffold with TODOs

**Target SDK repo not accessible:**
- Check `gh auth status`
- Suggest `gh auth login`
- Cannot proceed without access

**Linting fails:**
- Show linting errors to user
- Attempt auto-fix (`--fix` flags)
- If cannot fix automatically, ask user for guidance

**Tests fail:**
- Show test failures
- Attempt to debug and fix
- If cannot fix, create implementation with failing tests and note in output

**Complex implementation:**
- For very complex features, generate scaffold with detailed TODOs
- Mark areas needing manual implementation
- Provide implementation notes in commit message

### Language-Specific Notes

When generating code, study the target SDK's existing code to understand conventions. Claude already knows language idioms - focus on SDK-specific patterns:

**Key areas to analyze:**
- Naming conventions (found via Grep/Glob on similar features)
- Linting tools and configuration files (`.eslintrc`, `pyproject.toml`, `.rubocop.yml`, etc.)
- Test framework and patterns (look at existing test files)
- Type annotation style (if applicable)
- Error handling patterns

**Common tooling by language:**
- Python: black, ruff, mypy, pytest
- JavaScript/TypeScript: eslint, prettier, jest/vitest
- Go: gofmt, golangci-lint, standard testing package
- Ruby: rubocop, rspec
- Java: checkstyle, junit
- C#/.NET: dotnet format, xunit/nunit


