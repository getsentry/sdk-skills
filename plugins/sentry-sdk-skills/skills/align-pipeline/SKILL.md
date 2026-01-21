---
name: sdk-align-pipeline
description: Align a feature across multiple Sentry SDKs automatically. Use when implementing a feature in many SDKs, rolling out alignment, or running full SDK workflow. Keywords: align SDKs, implement across SDKs, SDK workflow, multiple SDKs.
argument-hint: [develop-doc-url]
allowed-tools: Read Write Bash Skill AskUserQuestion TodoWrite
compatibility: Requires sdk-* skills, gh CLI, optional Linear MCP.
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
2. **`linear-initiative`** - Create Linear projects for selected SDK teams
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

Or provide the develop doc URL and reference implementation when prompted:

User: /sdk-align-pipeline
Skill: What is the develop doc URL?
User: https://github.com/getsentry/sentry-docs/pull/11912
Skill: Which SDK has the reference implementation? (optional)
User: javascript PR #16313
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

Display the status report from `sdk-feature-status` to the user, showing:
- Feature name and alignment scope
- Count of implemented, needs review, and not implemented SDKs
- List of SDKs needing implementation

### Step 3: Confirm Target SDKs

Use AskUserQuestion to ask which SDKs to implement. Present options:
- All SDKs needing implementation
- Select specific SDKs (provide list)
- Cancel

If user wants to select specific SDKs, use AskUserQuestion with multiSelect enabled to let them choose from the list of SDKs needing implementation.

### Step 4: Create Linear Tracking

Use AskUserQuestion to ask if they want to create Linear projects for tracking. Explain that Linear projects help teams track their work and link to the SDK alignment initiative.

**If yes:**
Invoke `linear-initiative` skill with the feature name and selected SDK teams. The skill will create a project for each team and link them to the initiative.

**If no:**
Skip Linear tracking, continue without it.

### Step 5: Generate Implementations

For each selected SDK, invoke `sdk-feature-generate` with the SDK name. Show progress indicating which SDK is currently being processed (e.g., "1/4") and inform the user as each completes.

**Handle results:**
- Track successful generations (linting and tests passed)
- Track partial successes (generated but with linting/test failures)
- Track complete failures (could not generate)
- Continue with next SDK regardless of individual failures

After all generations complete, summarize results and use AskUserQuestion to ask if they want to continue to PR creation. If no, stop and inform the user they can manually run `sdk-feature-pr` later for review.

### Step 6: Create Pull Requests

For each successfully generated implementation (skip those with test/linting failures), invoke `sdk-feature-pr` with the SDK name. Show progress indicating which SDK PR is being created and inform the user as each completes with the PR URL.

**Handle results:**
- Track successful PR creations with PR number and URL
- Track failures (branch not pushed, auth issues, etc.)
- Continue with remaining SDKs regardless of individual failures

### Step 7: Update Linear Projects

If Linear projects were created, update each SDK team's project with its corresponding PR link using Linear MCP tools (create comment or update description).

### Step 8: Final Summary

Display a complete summary of the pipeline execution including:

**Feature Information:**
- Feature name and alignment scope

**Status Check Results:**
- Count of SDKs already implemented, needing review, and selected for implementation

**Linear Tracking (if created):**
- Number of projects created
- Confirmation they're linked to the initiative

**Implementations Generated:**
- List each SDK with success/failure status
- Include branch names and any warnings (linting/test failures)

**Pull Requests Created:**
- List each PR with number and URL
- Note any SDKs skipped due to failures

**Manual Review Needed (if applicable):**
- List SDKs requiring fixes before PR creation
- Provide specific commands or guidance for fixing issues

**Next Steps:**
- Suggest reviewing PRs
- Note any manual fixes needed
- Reference Linear projects for tracking
- Mention updating develop docs if needed

Inform the user that context has been saved to `.sdk-align/` directory for future reference.

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

### Progress Tracking

**Use TodoWrite throughout the workflow:**

Create a todo list at the start with the main workflow steps:
1. Run status check
2. Confirm target SDKs
3. Create Linear tracking (if requested)
4. Generate implementations for each SDK
5. Create PRs for each SDK
6. Update Linear with PR links

Mark each step as in_progress when starting and completed when finished. For SDK-specific tasks (generation, PRs), add individual todos for each SDK and track them separately.

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
- Consider invoking multiple `sdk-feature-generate` skills in parallel if the system supports it
- Provide progress updates showing which SDKs are being processed
- Inform the user they can stop and resume later if needed

### Resuming Failed Runs

If pipeline fails mid-run, the skill can detect and resume from the partial state by checking for `.sdk-align/` directory with existing context.

When invoked again and partial context is found:
- Inform the user about what was completed, what failed, and what remains
- Use AskUserQuestion to ask if they want to resume from where it left off
- If confirmed, read context from `.sdk-align/` files and resume from last successful step


