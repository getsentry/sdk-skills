---
name: sdk-align-pipeline
description: Orchestrator skill that runs the complete SDK feature alignment workflow end-to-end. Checks status, creates Linear tracking, generates code for selected SDKs, and creates PRs. Coordinates sdk-feature-status, sdk-linear-track, sdk-feature-generate, and sdk-feature-pr skills.
model: sonnet
allowed-tools: Read Write Bash Skill AskUserQuestion
compatibility: Requires all sdk-* skills, gh CLI, Linear MCP, and SDK-specific tooling.
---

# SDK Alignment Pipeline Orchestrator

End-to-end orchestrator for SDK feature alignment workflow.

## When to Use This Skill

Use this skill when:
- You want to run the complete SDK alignment workflow from start to finish
- You have a develop doc and want to align it across multiple SDKs
- You prefer automation over running each skill manually
- You want a guided, interactive alignment process

## What This Skill Does

This orchestrator coordinates all SDK alignment skills:

1. **`sdk-feature-status`** - Check implementation status across all SDKs
2. **`sdk-linear-track`** - Create Linear project and issue for tracking
3. **`sdk-feature-generate`** - Generate code for each selected SDK
4. **`sdk-feature-pr`** - Create PRs for each implementation

It manages the workflow, handles errors, and provides progress updates.

## Process

### Step 1: Gather Initial Input

Ask user for:
- **Develop doc** (PR URL, commit SHA, or file path) - Required
- **Reference implementation** (SDK name + optional PR URL) - Optional, will search if not provided
- **Linear issue ID** (if existing tracking) - Optional, will search/create if not provided

**Example:**
```
User: /sdk-align-pipeline https://github.com/getsentry/sentry-docs/pull/11912

Or:

User: /sdk-align-pipeline https://github.com/getsentry/sentry-docs/pull/11912 --reference javascript#16313
```

### Step 2: Run Status Check

Invoke `sdk-feature-status` skill:

```
Invoking sdk-feature-status...

Checking implementation status across all SDKs...
```

**`sdk-feature-status`** will:
- Fetch and analyze develop doc
- Find reference implementation (if not provided)
- Check all SDKs for existing implementations
- Generate status report
- Save context to `.sdk-align/`

**Display status table to user:**
```
SDK Feature Status Report
=========================
Feature: Strict Trace Propagation
Alignment Scope: Core

Summary:
  ✅ Implemented: 2 SDKs
  ⚠️  Needs Review: 1 SDK
  ❌ Not Implemented: 14 SDKs
```

### Step 3: Confirm Target SDKs

Present SDKs needing implementation and ask user to confirm:

```
The following SDKs need implementation:
  ruby, go, java, dotnet, rust, android, cocoa, react-native,
  flutter, unity, unreal, native, elixir, kotlin

Which SDKs would you like to implement?

a) All 14 SDKs
b) Select specific SDKs
c) Cancel and exit
```

**If user selects (b):**
Show interactive checklist:
```
Select SDKs to implement:
  [x] ruby
  [x] go
  [x] java
  [x] dotnet
  [ ] rust
  [ ] android
  [ ] cocoa
  [ ] react-native
  [ ] flutter
  [ ] unity
  [ ] unreal
  [ ] native
  [ ] elixir
  [ ] kotlin

Continue with 4 selected SDKs? (y/n)
```

**Final SDK list** is confirmed by user.

### Step 4: Create Linear Tracking

Ask user if they want to create Linear tracking:

```
Create Linear project and issue to track this alignment work?

Linear will help track:
- Overall progress across SDKs
- PR links as they're created
- Implementation status

Create Linear tracking? (y/n)
```

**If yes:**
Invoke `sdk-linear-track` skill:

```
Invoking sdk-linear-track...

Creating Linear tracking...
✅ Project created: [SDK Alignment] Strict Trace Propagation
✅ Issue created: GSD-1234
```

**If no:**
Skip Linear tracking, continue without it.

### Step 5: Generate Implementations

For each selected SDK, invoke `sdk-feature-generate`:

**Show progress:**
```
Generating implementations for 4 SDKs...

[1/4] Generating code for Ruby SDK...
```

**Invoke generate skill:**
```
Invoking sdk-feature-generate ruby...
```

**Handle results:**
- ✅ **Success**: Implementation generated, linting passed, tests passed
- ⚠️ **Partial success**: Implementation generated, but linting/tests failed
- ❌ **Failure**: Could not generate implementation

**Continue with next SDK** regardless of individual failures.

**Progress updates:**
```
[1/4] Ruby ✅ Generated successfully
[2/4] Go ✅ Generated successfully
[3/4] Java ⚠️  Generated with warnings (1 test failed)
[4/4] .NET ✅ Generated successfully
```

**After all generations, show summary:**
```
Implementation Summary:
  ✅ Successful: 3 SDKs (Ruby, Go, .NET)
  ⚠️  Warnings: 1 SDK (Java - 1 test failure)
  ❌ Failed: 0 SDKs

Continue to PR creation? (y/n)
```

**If user selects no:**
- Stop here
- User can review implementations locally
- User can manually run `sdk-feature-pr` later

### Step 6: Create Pull Requests

For each successfully generated implementation, invoke `sdk-feature-pr`:

**Show progress:**
```
Creating PRs for 3 SDKs (skipping Java due to test failures)...

[1/3] Creating PR for Ruby SDK...
```

**Invoke PR skill:**
```
Invoking sdk-feature-pr ruby...
```

**Handle results:**
- ✅ **Success**: PR created
- ❌ **Failure**: PR creation failed (branch not pushed, auth issue, etc.)

**Progress updates:**
```
[1/3] Ruby ✅ PR #8888 created
[2/3] Go ✅ PR #7777 created
[3/3] .NET ✅ PR #6666 created
```

### Step 7: Update Linear Issue

If Linear tracking was created, update the issue with PR links:

```
Updating Linear issue with PR links...

Invoking sdk-linear-track --update-issue GSD-1234...

✅ Linear issue updated with 3 PR links
```

### Step 8: Final Summary

Display complete summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SDK Alignment Pipeline Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Feature: Strict Trace Propagation
Alignment Scope: Core

Status Check:
  ✅ 2 SDKs already implemented
  ⚠️  1 SDK needs review
  🎯 4 SDKs selected for implementation

Linear Tracking:
  ✅ Project: [SDK Alignment] Strict Trace Propagation
  ✅ Issue: GSD-1234
  📊 https://linear.app/getsentry/issue/GSD-1234

Implementations Generated:
  ✅ Ruby - Branch: feat/strict-trace-continuation
  ✅ Go - Branch: feat/strict-trace-continuation
  ⚠️  Java - Branch: feat/strict-trace-continuation (1 test failure)
  ✅ .NET - Branch: feat/strict-trace-continuation

Pull Requests Created:
  ✅ Ruby #8888 - https://github.com/getsentry/sentry-ruby/pull/8888
  ✅ Go #7777 - https://github.com/getsentry/sentry-go/pull/7777
  ✅ .NET #6666 - https://github.com/getsentry/sentry-dotnet/pull/6666

Manual Review Needed:
  ⚠️  Java - Fix test failure before creating PR
     Run: cd /tmp/sentry-java && pytest tests/test_strict_trace.py

Next Steps:
  1. Review PRs and address any feedback
  2. Fix Java test failure and create PR manually
  3. Track progress in Linear: GSD-1234
  4. Update develop docs if needed

Context saved to .sdk-align/ directory.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Guidelines

### User Interaction

**Be Interactive:**
- Ask for confirmation at key decision points
- Don't blindly process everything
- Let user guide the workflow

**Key confirmation points:**
1. After status check: Which SDKs to implement?
2. Before Linear creation: Create tracking?
3. After generation: Continue to PR creation?
4. If errors occur: Skip or retry?

**Provide Progress Updates:**
- Show current step (e.g., "[2/4] Generating code for Go...")
- Display results as they complete
- Don't wait until end to show all results

**Handle Errors Gracefully:**
- If one SDK fails, continue with others
- Collect all errors and show at end
- Provide actionable next steps

### Workflow Flexibility

Allow users to:
- **Skip steps**: "Don't create Linear tracking"
- **Pause and resume**: "Stop after generation, I'll create PRs manually"
- **Partial runs**: "Only implement for Python and Go"
- **Retry failures**: "Retry Java after I fix the test"

### Error Handling

**Common errors and handling:**

**Develop doc not accessible:**
- Display error
- Ask user to provide develop doc content or alternative URL
- Allow manual input

**Reference implementation not found:**
- Ask user which SDK should be reference
- Or continue with develop doc only
- Generate best-effort implementations

**SDK generation fails:**
- Mark as failed
- Continue with other SDKs
- Show error details at end
- Provide manual fix instructions

**PR creation fails:**
- Mark as failed
- Continue with other SDKs
- User can manually create PR later using `sdk-feature-pr`

**Linear MCP unavailable:**
- Skip Linear tracking
- Continue with rest of workflow
- Inform user to create Linear issue manually

### Batch Processing Optimization

For large SDK sets (10+ SDKs):
- Generate code for SDKs in parallel where possible
- Show progress bar or percentage
- Allow user to cancel and resume later

**Example parallel generation:**
```
Generating implementations (parallel)...

Ruby   ████████████████░░░░  75% (linting)
Go     ████████████████████ 100% ✅
Java   ████████░░░░░░░░░░░░  40% (generating tests)
.NET   ████░░░░░░░░░░░░░░░░  20% (analyzing SDK)
```

### Resuming Failed Runs

If pipeline fails mid-run, user can resume:

```
User: /sdk-align-pipeline --resume

Skill:
Detected partial run in .sdk-align/ directory:
  ✅ Status check completed
  ✅ Linear tracking created (GSD-1234)
  ✅ 2/4 implementations generated (Ruby, Go)
  ❌ Java generation failed
  ⏳ .NET not started

Resume from Java? (y/n)
```

Read context from `.sdk-align/` and resume from last successful step.

## Example Workflows

### Full Automated Run

```
User: /sdk-align-pipeline https://github.com/getsentry/sentry-docs/pull/11912

1. Status check runs
2. User selects: "Implement for Ruby, Go, Java, .NET"
3. User confirms: "Yes, create Linear tracking"
4. Implementations generated for all 4
5. PRs created for all 4
6. Linear updated
7. Summary displayed
```

### Partial Run with Manual Review

```
User: /sdk-align-pipeline https://github.com/getsentry/sentry-docs/pull/11912

1. Status check runs
2. User selects: "Implement for Python only"
3. User confirms: "No Linear tracking"
4. Python implementation generated
5. User prompted: "Continue to PR creation?"
6. User: "No, I'll review first"
7. Summary displayed, user reviews code locally
8. Later: User runs /sdk-feature-pr python manually
```

### Resume After Failure

```
User: /sdk-align-pipeline https://github.com/getsentry/sentry-docs/pull/11912

1. Status check runs
2. User selects: "All 14 SDKs"
3. Implementations generated for 10/14
4. Go generation fails (auth error)
5. User fixes auth, runs:
   /sdk-align-pipeline --resume
6. Pipeline resumes from failed SDK
7. Completes remaining SDKs
```

## Advanced Options

**Command-line style arguments:**

```
--reference <sdk>#<pr-num>   Specify reference implementation
--sdks <list>                 Specific SDKs to implement
--skip-linear                 Don't create Linear tracking
--skip-pr                     Don't create PRs (generation only)
--resume                      Resume from partial run
--dry-run                     Show what would be done, don't execute
```

**Examples:**
```
/sdk-align-pipeline https://github.com/getsentry/sentry-docs/pull/11912 --reference javascript#16313 --sdks python,go,ruby

/sdk-align-pipeline https://github.com/getsentry/sentry-docs/pull/11912 --skip-pr

/sdk-align-pipeline --resume
```

## References

- [SDK Feature Status Skill](../sdk-feature-status/SKILL.md)
- [SDK Feature Generate Skill](../sdk-feature-generate/SKILL.md)
- [SDK Feature PR Skill](../sdk-feature-pr/SKILL.md)
- [SDK Linear Track Skill](../sdk-linear-track/SKILL.md)
- [Sentry SDK Philosophy](https://develop.sentry.dev/sdk/philosophy/)
