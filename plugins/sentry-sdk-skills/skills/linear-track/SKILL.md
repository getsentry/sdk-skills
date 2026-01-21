---
name: sdk-linear-track
description: Create or update Linear project and issue for SDK feature alignment tracking. Links to "SDK Projects to Align Across SDKs" initiative, tracks implementation progress across SDKs, and updates with PR links. Requires Linear MCP server.
model: sonnet
allowed-tools: Read Write Bash AskUserQuestion TodoWrite
compatibility: Requires Linear MCP server configured and authenticated.
---

# SDK Linear Tracking

Create and manage Linear projects and issues for SDK feature alignment work.

## When to Use This Skill

Use this skill when:
- Starting SDK alignment work and need Linear tracking
- You have a develop doc and want to create a Linear issue
- You need to update an existing Linear issue with PR links
- You want to track SDK implementation progress in Linear

## Shared Context System

This skill can:
- **Read** from `.sdk-align/context.json` (develop doc, reference impl)
- **Read** from `.sdk-align/status-report.json` (SDK implementation status)
- **Read** from `.sdk-align/prs.json` (created PRs to link)
- **Write** Linear issue ID to `.sdk-align/context.json`
- **Work standalone** if all information is provided directly

## Process

### Step 1: Gather Required Information

**Required inputs:**
- Feature name (e.g., "Strict Trace Propagation")

**Optional inputs (will be read from context if not provided):**
- Develop doc URL
- Reference implementation PR URL
- Target SDKs list
- Existing Linear issue ID (to update instead of create)

**Read from shared context:**

Use the Read tool to check for and read:
- `.sdk-align/context.json` (feature context)
- `.sdk-align/status-report.json` (SDK status)
- `.sdk-align/prs.json` (created PRs)

**If context files don't exist:**
Ask user for:
1. Feature name (required)
2. Develop doc URL (required)
3. Reference implementation (optional)
4. Target SDKs (optional, will use all SDKs if not specified)

### Step 2: Search for Existing Linear Issue

Before creating, search Linear for existing issues:

**Search strategies:**
1. Search by feature name in SDK teams
2. Search by develop doc URL in issue descriptions
3. Search in "SDK Projects to Align Across SDKs" initiative

**Using Linear MCP:**
```
Search Linear issues:
- Query: "<feature-name> SDK"
- Teams: Filter to SDK teams only
- Initiative: "SDK Projects to Align Across SDKs"
- Limit: 10 results
```

**Present results to user:**
```
Found potential existing issues:
1. [GSD-1234] Align Strict Trace Propagation across SDKs - Created 2025-06-15
2. [GSD-5678] Trace propagation improvements - Created 2025-05-20

Do you want to:
a) Use existing issue GSD-1234
b) Use existing issue GSD-5678
c) Create a new issue
```

If user selects existing issue, skip to Step 5 (Update Issue).

### Step 3: Create Linear Project

Create a Linear project for internal tracking:

**Project details:**
- **Name**: `[SDK Alignment] <Feature Name>`
- **Description**: Include develop doc link, reference implementation link, alignment scope
- **Status**: "In Progress"
- **Access**: Private (internal team tracking)

**Link to initiative:**
- **Initiative**: "SDK Projects to Align Across SDKs"
- **Parent Initiative**: "Great Data Capture (SDKs)" (if applicable)

**Using Linear MCP:**
```
Create project:
{
  "name": "[SDK Alignment] Strict Trace Propagation",
  "description": "Track implementation of Strict Trace Propagation across all Sentry SDKs.\n\nDevelop Doc: https://github.com/getsentry/sentry-docs/pull/11912\nReference: https://github.com/getsentry/sentry-javascript/pull/16313\nAlignment Scope: Core",
  "status": "in_progress",
  "initiativeId": "<SDK_PROJECTS_INITIATIVE_ID>"
}
```

**Track project ID** for linking the issue.

### Step 4: Create Linear Issue

Create a public-facing Linear issue within the project:

**Issue details:**
- **Title**: `Align <Feature Name> across SDKs`
- **Description**: Public-friendly explanation with links
- **Project**: Link to project created in Step 3
- **Team**: Assign to appropriate SDK team (or Platform/SDK team)
- **Labels**: `sdk-alignment`, feature-specific labels
- **Priority**: Ask user or default to Medium
- **Status**: "Backlog" or "In Progress"

**Description template:**

```markdown
## Overview

Implement <Feature Name> across all applicable Sentry SDKs to ensure consistent behavior.

## Specification

The feature is defined in the develop docs:
<develop-doc-url>

## Reference Implementation

The reference implementation is in the <SDK> SDK:
<reference-pr-url>

## Alignment Scope

**<Core|Domain|Platform>** - <Brief explanation of scope>

## Target SDKs

The following SDKs need this feature:
- [ ] Python
- [ ] JavaScript
- [ ] Ruby
- [ ] PHP
- [ ] Go
- [ ] Java
- [ ] .NET
- [ ] Rust
- [ ] Android
- [ ] iOS/macOS
- [ ] React Native
- [ ] Flutter
- [ ] Unity
- [ ] Unreal
- [ ] Elixir
- [ ] Kotlin Multiplatform

## Implementation Status

<Will be updated as PRs are created>

## Related

- Develop Doc: <url>
- Reference Implementation: <url>
```

**Pre-fill checkboxes based on status report:**
If `.sdk-align/status-report.json` exists, mark implemented SDKs:
- [x] JavaScript (✅ Implemented - PR #16313)
- [x] Python (✅ Implemented - PR #5178)
- [ ] Go (❌ Not Implemented)

**Using Linear MCP:**
```
Create issue:
{
  "title": "Align Strict Trace Propagation across SDKs",
  "description": "<markdown-description>",
  "projectId": "<PROJECT_ID>",
  "teamId": "<SDK_TEAM_ID>",
  "labels": ["sdk-alignment", "tracing"],
  "priority": "medium",
  "status": "in_progress"
}
```

**Track issue ID** for future updates and context saving.

### Step 5: Update Issue with PR Links

If PRs have been created (from `.sdk-align/prs.json`), add them to the issue:

**Add comment with PR links:**

```markdown
## Implementation Progress

The following PRs have been created:

- **Python**: https://github.com/getsentry/sentry-python/pull/9999
- **Go**: https://github.com/getsentry/sentry-go/pull/8888
- **Java**: https://github.com/getsentry/sentry-java/pull/7777
```

**Update issue description:**
Update the checklist in the description to mark PRs:
- [x] Python - PR #9999
- [x] Go - PR #8888
- [x] Java - PR #7777

**Using Linear MCP:**
```
Add comment:
{
  "issueId": "<ISSUE_ID>",
  "body": "<comment-markdown>"
}

Update issue:
{
  "issueId": "<ISSUE_ID>",
  "description": "<updated-description-with-pr-links>"
}
```

### Step 6: Update Shared Context

Write Linear issue ID to `.sdk-align/context.json`:

```json
{
  "version": "1.0",
  "feature": {
    "name": "Strict Trace Propagation",
    "developDoc": {
      "url": "https://github.com/getsentry/sentry-docs/pull/11912",
      "commit": "399f5292fdf53a0b78f7de01978c7b329d2ab717"
    },
    "referenceImplementation": {
      "sdk": "javascript",
      "pr": "https://github.com/getsentry/sentry-javascript/pull/16313"
    },
    "alignmentScope": "Core",
    "linearIssue": "GSD-1234",
    "linearProject": "GSD-PROJ-123"
  }
}
```

Update the context file with Linear tracking information.

### Step 7: Output Summary

Display to user:
- ✅ Linear tracking created
- Project: `[SDK Alignment] <Feature Name>` (<project-url>)
- Issue: `<Issue Title>` (<issue-url>)
- Issue ID: `GSD-1234`
- Initiative: Linked to "SDK Projects to Align Across SDKs"
- PRs linked: X PRs
- Context saved to `.sdk-align/context.json`

Show the Linear issue URL prominently.

## Guidelines

### Progress Tracking

**Use TodoWrite to track Linear tracking creation:**

Create a todo list with the following steps:
1. Gather required information
2. Search for existing Linear issue
3. Create Linear project (if creating new)
4. Create Linear issue (if creating new)
5. Update issue with PR links (if applicable)
6. Update shared context

Mark each step as in_progress when starting and completed when finished. This helps users understand the multi-step process of creating or updating Linear tracking.

### Linear Issue Best Practices

1. **Clear Title**
   - Use "Align X across SDKs" format
   - Makes it easy to identify alignment issues
   - Searchable and consistent

2. **Public-Friendly Description**
   - Avoid internal jargon
   - Explain what the feature does (not just implementation details)
   - Include links to specs

3. **Checkbox Tracking**
   - One checkbox per SDK
   - Update as PRs are created/merged
   - Visual progress tracking

4. **Regular Updates**
   - Add comments when PRs are created
   - Update when PRs are merged
   - Close issue when all SDKs are aligned

5. **Proper Labeling**
   - `sdk-alignment` - Identifies alignment work
   - Feature-specific labels (e.g., `tracing`, `performance`)
   - Domain labels if applicable (`backend`, `frontend`, `mobile`)

### Linear Project Structure

**Project vs Issue:**
- **Project**: Internal tracking, private, detailed progress
- **Issue**: Public-facing, high-level, linked to project

**Why both?**
- Project: Engineers track detailed work
- Issue: Stakeholders see overall progress
- Project can have milestones, internal notes
- Issue is clean, public-appropriate

### Handling Edge Cases

**Linear MCP not available:**
- Inform user that Linear MCP is required
- Provide manual instructions to create issue
- Show suggested issue title and description
- Cannot proceed with automated creation

**Initiative not found:**
- Search for "SDK Projects to Align Across SDKs"
- If not found, ask user for initiative ID
- Or create issue without initiative link
- Inform user to link manually

**Search returns many results:**
- Show top 5 most relevant
- Allow user to select or create new
- Sort by creation date (most recent first)

**Issue already exists:**
- Offer to update existing issue instead
- Add new PR links as comment
- Update description with new PRs
- Don't create duplicate

**Permission errors:**
- Check Linear authentication
- Suggest re-authenticating Linear MCP
- Provide manual fallback instructions

### Multi-Project Tracking

For very large alignment efforts, consider:
- One parent project for overall tracking
- Child projects per SDK or per domain
- Link all to main alignment issue
- Update parent project as children complete

## Example Workflow

```
User: /sdk-linear-track "Strict Trace Propagation"

Skill:
1. Reads context from .sdk-align/context.json
2. Reads status from .sdk-align/status-report.json
3. Searches Linear for existing issues
4. No existing issue found
5. Creates project: "[SDK Alignment] Strict Trace Propagation"
6. Links to "SDK Projects to Align Across SDKs" initiative
7. Creates issue: "Align Strict Trace Propagation across SDKs"
8. Pre-fills checklist: 2 implemented, 14 not implemented
9. Reads .sdk-align/prs.json
10. Finds 3 PRs created (Python, Go, Java)
11. Adds comment with PR links
12. Updates issue description with PRs
13. Writes Linear IDs to context.json

User sees:
  ✅ Linear tracking created

  Project: [SDK Alignment] Strict Trace Propagation
  https://linear.app/getsentry/project/gsd-proj-123

  Issue: Align Strict Trace Propagation across SDKs
  https://linear.app/getsentry/issue/GSD-1234

  Issue ID: GSD-1234
  Initiative: SDK Projects to Align Across SDKs ✓
  PRs linked: 3 PRs

  Context saved for other sdk-* skills.
```

## References

- [Linear API Documentation](https://linear.app/developers)
- [Linear MCP Integration](https://linear.app/docs/mcp)
- [Sentry Engineering Practices](https://develop.sentry.dev/engineering-practices/)
