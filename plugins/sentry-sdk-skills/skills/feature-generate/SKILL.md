---
name: sdk-feature-generate
description: Generate idiomatic implementation code for ONE Sentry SDK based on develop docs and reference implementation. Analyzes SDK patterns, follows conventions, generates tests, runs linting, and commits changes. Works standalone or with context from sdk-feature-status.
model: sonnet
allowed-tools: Read Grep Glob Bash Write Edit Task
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

Based on reference implementation + target SDK patterns, generate code:

#### Core Implementation Files

Adapt reference implementation to target SDK:

**Example: Python (from JavaScript reference)**

JavaScript reference:
```javascript
// packages/core/src/tracing/strictTrace.ts
export function shouldContinueTrace(incomingOrg?: string): boolean {
  const options = getClient()?.getOptions();
  if (!options?.strictTraceContinuation) {
    return true;
  }

  const configuredOrg = options.org || parseOrgFromDsn(options.dsn);
  return incomingOrg === configuredOrg;
}
```

Python adaptation:
```python
# sentry_sdk/tracing/strict_trace.py
def should_continue_trace(incoming_org=None):
    # type: (Optional[str]) -> bool
    """
    Determine if trace should be continued based on org ID.

    Args:
        incoming_org: Organization ID from incoming baggage

    Returns:
        True if trace should be continued, False otherwise
    """
    client = sentry_sdk.get_client()
    options = client.options if client else {}

    if not options.get("strict_trace_continuation", False):
        return True

    configured_org = options.get("org") or _parse_org_from_dsn(options.get("dsn"))
    return incoming_org == configured_org


def _parse_org_from_dsn(dsn):
    # type: (Optional[str]) -> Optional[str]
    """Extract org ID from DSN (e.g., 'o1' from 'https://key@o1.ingest.sentry.io/project')."""
    if not dsn:
        return None

    import re
    match = re.search(r'@(o\d+)\.', dsn)
    return match.group(1) if match else None
```

Key adaptations:
- **Naming**: `shouldContinueTrace` → `should_continue_trace`
- **Type hints**: JSDoc → Python type comments
- **Docstrings**: Added Python docstring format
- **Idioms**: Used Python's `get()` method, optional parameters

#### Configuration/Options

Add config option to SDK's configuration:

**Python** (`sentry_sdk/consts.py` or similar):
```python
# Add to default options
DEFAULT_OPTIONS = {
    # ... existing options ...
    "strict_trace_continuation": False,
    "org": None,
}
```

**JavaScript** (`packages/types/src/options.ts`):
```typescript
export interface ClientOptions {
  // ... existing options ...
  strictTraceContinuation?: boolean;
  org?: string;
}
```

#### Tests

Generate tests mirroring reference implementation tests:

**JavaScript reference test**:
```javascript
describe('strictTraceContinuation', () => {
  it('continues trace when disabled', () => {
    const result = shouldContinueTrace('o123');
    expect(result).toBe(true);
  });

  it('blocks trace when org mismatch', () => {
    setOptions({ strictTraceContinuation: true, org: 'o456' });
    const result = shouldContinueTrace('o123');
    expect(result).toBe(false);
  });
});
```

**Python adaptation**:
```python
# tests/test_strict_trace.py
import pytest
from sentry_sdk.tracing.strict_trace import should_continue_trace


def test_continues_trace_when_disabled(sentry_init):
    """Trace continues when strict_trace_continuation is False (default)."""
    sentry_init()
    assert should_continue_trace("o123") is True


def test_blocks_trace_when_org_mismatch(sentry_init):
    """Trace blocked when org IDs don't match."""
    sentry_init(strict_trace_continuation=True, org="o456")
    assert should_continue_trace("o123") is False


def test_continues_trace_when_org_matches(sentry_init):
    """Trace continues when org IDs match."""
    sentry_init(strict_trace_continuation=True, org="o123")
    assert should_continue_trace("o123") is True


def test_parses_org_from_dsn(sentry_init):
    """Org ID parsed from DSN if not explicitly configured."""
    sentry_init(
        dsn="https://key@o789.ingest.sentry.io/123",
        strict_trace_continuation=True
    )
    assert should_continue_trace("o789") is True
    assert should_continue_trace("o999") is False
```

Key adaptations:
- **Framework**: jest → pytest
- **Assertions**: `expect().toBe()` → `assert ... is ...`
- **Fixtures**: Used pytest `sentry_init` fixture
- **Test names**: Descriptive Python function names

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

**Python:**
- Use type hints (PEP 484)
- Follow PEP 8 naming conventions
- Use `black` for formatting (line length 88)
- Tests with pytest, use fixtures

**JavaScript/TypeScript:**
- Use TypeScript types for all new code
- Follow existing ESLint rules
- Use jest or vitest for tests
- Prefer async/await over promises

**Go:**
- Follow Go conventions (gofmt, golint)
- Exported names start with uppercase
- Use table-driven tests
- Error handling: return errors, don't panic

**Ruby:**
- Follow RuboCop rules
- Use `snake_case` everywhere
- RSpec for tests with `describe`/`context`/`it`
- Use idiomatic Ruby (blocks, enumerables)

**Java:**
- Follow Google Java Style
- Use JUnit for tests
- Proper exception handling
- Use Optional for nullable values

**C#/.NET:**
- Follow C# conventions
- Use async/await for async code
- xUnit or NUnit for tests
- LINQ where appropriate

## Example Workflow

```
User: /sdk-feature-generate python

Skill:
1. Reads context from .sdk-align/context.json
2. Finds feature: "Strict Trace Propagation"
3. Fetches JavaScript reference PR #16313
4. Clones sentry-python repository
5. Analyzes Python conventions (snake_case, pytest, black)
6. Generates:
   - sentry_sdk/tracing/strict_trace.py
   - Updates sentry_sdk/consts.py
   - tests/test_strict_trace.py
7. Runs black, ruff, mypy, pytest
8. All checks pass ✓
9. Commits changes
10. Writes to .sdk-align/implementations.json

User sees:
  ✅ Implementation generated for python
  Branch: feat/strict-trace-continuation
  Files: 3 changed
  Commit: abc123
  Linting: Passed ✓
  Tests: Passed ✓ (5 tests)

  Next: Run /sdk-feature-pr python
```

## References

- [Sentry SDK Philosophy](https://develop.sentry.dev/sdk/philosophy/)
- [Python SDK Contributing](https://github.com/getsentry/sentry-python/blob/master/CONTRIBUTING.md)
- [JavaScript SDK Contributing](https://github.com/getsentry/sentry-javascript/blob/develop/CONTRIBUTING.md)
