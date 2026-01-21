# Usage Guide

## Aligning a Feature Across SDKs

**Step 1: Check implementation status**

```bash
/sdk-feature-status https://github.com/getsentry/sentry-docs/pull/12345
```

This analyzes which SDKs have implemented the feature and creates `.sdk-align/` context for other skills.

**Step 2: Start with 1-2 SDKs for testing**

```bash
/sdk-feature-generate python
```

What happens:
- Asks if you have a local checkout or should clone to /tmp/
- Analyzes reference implementation and target SDK patterns
- Generates idiomatic code following SDK conventions
- Shows diff for review before committing
- Runs linting and tests with error recovery options
- Commits using sentry-skills:commit
- Asks: "Create pull request now?"
  - If yes: Invokes sdk-feature-pr automatically
  - If no: You can run /sdk-feature-pr later

Repeat for 1-2 SDKs to validate the approach.

**Step 3: Expand to remaining SDKs**

Once validated, repeat for other SDKs:

```bash
/sdk-feature-generate go
/sdk-feature-generate ruby
/sdk-feature-generate java
# etc.
```

Each implementation:
- Reads context from `.sdk-align/` (develop doc, reference PR)
- Generates idiomatic code for that SDK
- Optionally creates PR at the end

**Step 4: Track in Linear (optional)**

```bash
/linear-initiative
```

Creates Linear projects for SDK teams and links PRs for tracking.

## Deferred PR Creation

If you declined PR creation during generation:

```bash
/sdk-feature-pr python
```

This:
- Reads context from `.sdk-align/`
- Navigates to the repository
- Invokes sentry-skills:create-pr with alignment context
- Tracks the PR in `.sdk-align/prs.json`
- Updates Linear issue if configured

## Working with Local Repositories

When you run `/sdk-feature-generate`, it asks: "Do you have a local checkout?"

**If yes:**
- Provide path (e.g., `~/code/sentry-python`)
- Works in your local repo
- Asks before pushing to remote
- You can review/test before pushing

**If no:**
- Clones to `/tmp/<sdk-name>`
- Automatically pushes to prevent data loss
- Warning: changes lost if session ends before push

## Shared Context

Skills communicate via `.sdk-align/` directory:
- `context.json` - Feature details, develop doc, reference implementation
- `status-report.json` - SDK implementation status
- `implementations.json` - Generated code details per SDK
- `prs.json` - Created PR links per SDK

This enables running skills independently while maintaining context.

## Workflow Examples

### Iterative Development (Recommended)

```bash
# Check status
/sdk-feature-status <develop-doc-url>

# Generate + test first SDK
/sdk-feature-generate python
# → Review code, run additional tests

# Generate + test second SDK
/sdk-feature-generate go
# → Verify pattern works across languages

# Expand to remaining SDKs
/sdk-feature-generate ruby
/sdk-feature-generate java
# etc.

# Create Linear tracking
/linear-initiative
```

### Quick Workflow

```bash
# Check status and generate in sequence
/sdk-feature-status <develop-doc-url>
/sdk-feature-generate python  # Say "yes" to PR creation
/sdk-feature-generate go       # Say "yes" to PR creation
/sdk-feature-generate ruby     # Say "yes" to PR creation
```

### Deferred PR Workflow

```bash
# Generate code only, defer PR creation
/sdk-feature-generate python  # Say "no" to PR creation
/sdk-feature-generate go       # Say "no" to PR creation

# Review locally, run additional tests, manual fixes

# Create PRs when ready
/sdk-feature-pr python
/sdk-feature-pr go
```
