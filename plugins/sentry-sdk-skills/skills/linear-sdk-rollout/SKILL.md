---
name: linear-sdk-rollout
description: Creates Linear projects from an initiative or parent-child issues for cross-SDK rollouts. Use when rolling out features across multiple SDKs, creating SDK projects from initiative, setting up cross-SDK alignment work, creating parent-child issue hierarchy, or organizing SDK tasks.
allowed-tools: Read mcp__linear-server__query_data mcp__linear-server__list_teams mcp__linear-server__list_projects mcp__linear-server__create_project mcp__linear-server__get_project mcp__linear-server__update_project mcp__linear-server__create_issue AskUserQuestion
compatibility: Requires the Linear MCP server to be configured
---

# Linear SDK Rollout

Creates Linear projects or parent-child issues for rolling out work across SDK teams.

## Requirements

This skill requires the [Linear MCP server](https://github.com/linear/linear-mcp) to be configured.

## Parent Teams

These are the teams where a parent issue can be created (Path B only):

| Parent Team | Category |
|-------------|----------|
| Backend SDKs | Backend |
| JavaScript SDKs | JavaScript |
| Mobile SDKs | Mobile |
| SDKs | General (also used for Gaming) |

## SDK Teams

Only teams with Enabled=Yes are included when creating projects or issues.

| Team | Suffix | Category | Enabled |
|------|--------|----------|---------|
| sentry-android | Android | Mobile | Yes |
| sentry-capacitor | Capacitor | JavaScript, Mobile | Yes |
| sentry-cocoa | Apple | Mobile | Yes |
| sentry-cordova | Cordova | Mobile | Yes |
| sentry-dart | Dart/Flutter | Mobile | Yes |
| sentry-dotnet | .NET | Backend | Yes |
| sentry-electron | Electron | JavaScript | Yes |
| sentry-elixir | Elixir | Backend | Yes |
| sentry-go | Go | Backend | Yes |
| sentry-godot | Godot | Gaming | Yes |
| sentry-java | Java | Backend | Yes |
| sentry-javascript | JavaScript | JavaScript | Yes |
| sentry-kotlin-multiplatform | KMP | Mobile | Yes |
| sentry-laravel | Laravel | Backend | Yes |
| sentry-lynx | Lynx | Mobile | No |
| sentry-native | Native | Gaming | Yes |
| sentry-php | PHP | Backend | Yes |
| sentry-powershell | PowerShell | Backend | No |
| sentry-python | Python | Backend | Yes |
| sentry-react-native | React Native | JavaScript, Mobile | Yes |
| sentry-ruby | Ruby | Backend | Yes |
| sentry-rust | Rust | Backend | Yes |
| sentry-symfony | Symfony | Backend | Yes |
| sentry-unity | Unity | Gaming | Yes |
| sentry-unreal | Unreal | Gaming | Yes |

## Category to Parent Team Mapping

When selecting SDK teams by category in Path B, use this mapping to determine which parent team hosts the parent issue:

| Category | Parent Team |
|----------|-------------|
| Backend | Backend SDKs |
| JavaScript | JavaScript SDKs |
| Mobile | Mobile SDKs |
| Gaming | SDKs |

## Naming Rules

1. Remove any text in parentheses or brackets from the initiative name (if initiative-based)
2. Add suffix in square brackets for per-team artifacts

**Format**: `[Clean Name] [[Suffix]]`

Example: "SDK Handling HTTP 413 (Content Too Large)"
- Clean name: "SDK Handling HTTP 413"
- Per-team: "SDK Handling HTTP 413 [Python]", "SDK Handling HTTP 413 [Go]", etc.

## Instructions

### Step 1: Choose Output Type

Ask the user what they want to create:

- **Path A: Projects** — Create Linear projects linked to an initiative (one per SDK team)
- **Path B: Parent + child issues** — Create a parent issue in a parent SDK team with sub-task issues in individual teams

### Step 2: Determine Input Source

**Path A (Projects):**
- Requires a Linear initiative (ID, name, or URL)

**Path B (Issues):**
- **From a Linear initiative**: Provide an initiative ID, name, or URL
- **Manual input**: Provide a title and description directly

### Step 3: Gather Details

**If initiative-based (both paths):**

1. Fetch the initiative details using `mcp__linear-server__query_data`:
   ```
   Query: "Get initiative [identifier] with its name, description, targetDate, and any associated projects"
   ```
2. Present the initiative summary to the user:
   - Initiative name
   - Description/content
   - Target date (if set)
   - Any existing projects already linked (Path A)

**If manual input (Path B only):**

1. Ask the user for:
   - **Title**: The issue title (will be used for parent and sub-tasks)
   - **Description**: The issue description/body

### Step 4: Select SDK Teams

Ask the user which SDK teams should receive projects or sub-task issues. Present options:

- **All enabled teams** (all teams with Enabled=Yes from the table)
- **By category**: JavaScript, Backend, Mobile, or Gaming (filter by Category column, only enabled teams)
- **Individual selection**: Let user specify specific teams

If a team belongs to multiple categories (e.g., sentry-capacitor is both JavaScript and Mobile), include it if any of its categories match.

### Step 5 (Path B only): Select Parent Team

Ask the user which parent team should host the parent issue. Present the options from the Parent Teams table. If the user has already indicated a category (e.g., "create issues for all Backend SDKs"), infer the parent team from the Category to Parent Team Mapping.

### Step 6: Confirm Naming

**Path A (Projects):**
- Default: Use initiative name with team suffix (e.g., "Session Replay [Python]")
- No suffix: Use initiative name only (same name for all projects)
- Custom: Let user specify a custom naming pattern

**Path B (Issues):**
- Parent issue uses the clean name directly (no suffix)
- Sub-task issues add the team suffix in square brackets

### Step 7: Confirm Full Plan

Present a complete summary before creating anything:

**Path A:**
- Initiative being used
- List of projects to create with names and teams
- Total projects to create

**Path B:**
- Parent issue: title, parent team, description preview
- Sub-task issues: list of teams and their issue titles (with suffixes)
- Total issues to create: 1 parent + N sub-tasks

**Require explicit confirmation** before proceeding.

### Step 8: Create Artifacts

**Path A (Projects):**

For each selected team, create a project using `mcp__linear-server__create_project`:
- **name**: Use the initiative name with suffix from the SDK Teams table (if suffixes enabled)
- **team**: The team name
- **description**: "Have a look at the [initiative](<initiative-url>) for a full description of this project."
- **initiative**: Link to the source initiative
- **state**: Set to "Backlog" (use status ID `d4127831-7550-41f2-918c-d7699c6c7b95`)
- **targetDate**: If the initiative has a target date set, apply it to the project

**(Optional) Ask about creating issues in each project:**
- Warn the user: "Issues may be automatically synced to GitHub. Do you want to create an issue in each project?"
- **Require explicit confirmation** before proceeding
- If confirmed, ask for issue content preference:
  - **Full description**: Use the complete initiative description as the issue body
  - **Placeholder**: "Have a look at the [initiative](<initiative-url>) for a full description."
- Create one issue per project using `mcp__linear-server__create_issue`:
  - **title**: Same as project name (e.g., "SDK Handling HTTP 413 [Python]")
  - **team**: The team name
  - **project**: Link to the created project
  - **description**: Based on user's choice above

**Path B (Issues):**

1. Create the parent issue using `mcp__linear-server__create_issue`:
   - **title**: The clean issue title (no suffix)
   - **team**: The selected parent team name
   - **description**: The issue description (if initiative-based, prepend: "Related initiative: [initiative-name](<initiative-url>)\n\n[description]")

2. Record the returned issue ID for use as `parentId`.

3. For each selected SDK team, create a sub-task issue using `mcp__linear-server__create_issue`:
   - **title**: Issue title with team suffix (e.g., "SDK Handling HTTP 413 [Python]")
   - **team**: The individual SDK team name (e.g., "sentry-python")
   - **description**: Same description as the parent issue
   - **parentId**: The ID of the parent issue created above

### Step 9: Report Results

Present a summary of all created artifacts:
- Successfully created projects or issues with links
- Each sub-task issue with link and team (Path B)
- Any failures and reasons

## Example Usage

### Example 1: Projects from initiative (Path A)

User: "Create projects for the Session Replay initiative across all Mobile SDKs"

Response flow:
1. Fetch "Session Replay" initiative details
2. Confirm the initiative found is correct
3. Ask about naming preference (with suffix, no suffix, or custom)
4. Create projects with names like:
   - "Session Replay [Android]" (sentry-android)
   - "Session Replay [Capacitor]" (sentry-capacitor)
   - "Session Replay [Apple]" (sentry-cocoa)
   - "Session Replay [Dart/Flutter]" (sentry-dart)
   - etc.
5. Report created project links

### Example 2: Issues from initiative (Path B)

User: "Create a parent issue for the Session Replay initiative in Mobile SDKs"

Response flow:
1. Fetch "Session Replay" initiative details
2. Confirm the initiative found is correct
3. Parent team: Mobile SDKs
4. Ask which Mobile SDK teams should get sub-tasks
5. Confirm plan:
   - Parent: "Session Replay" in Mobile SDKs
   - Sub-tasks: "Session Replay [Android]", "Session Replay [Apple]", etc.
6. Create parent issue, then sub-tasks linked to it
7. Report all created issue links

### Example 3: Issues from manual input (Path B)

User: "Create a parent issue 'Implement Span Links' in Backend SDKs for Python, Ruby, and Go"

Response flow:
1. Title: "Implement Span Links", ask for description
2. Parent team: Backend SDKs
3. Selected teams: sentry-python, sentry-ruby, sentry-go
4. Confirm plan:
   - Parent: "Implement Span Links" in Backend SDKs
   - Sub-tasks: "Implement Span Links [Python]", "Implement Span Links [Ruby]", "Implement Span Links [Go]"
5. Create parent issue, then sub-tasks linked to it
6. Report all created issue links

## Notes

- Always confirm the full plan before creating any artifacts
- Check for existing projects to avoid duplicates (Path A)
- Projects are automatically linked to the initiative when the `initiative` parameter is provided (Path A)
- If a project already exists for a team under this initiative, skip it and notify the user (Path A)
- The parent issue must be created first to obtain its ID for linking sub-tasks (Path B)
- Sub-task issues live in the individual SDK team's space, NOT in the parent team (Path B)
- If creation fails for any item, continue with remaining teams and report failures at the end
- **GitHub Sync Warning**: Some teams have Linear-GitHub sync enabled. Issues created in Linear may automatically create corresponding GitHub issues. Always warn the user about this before creating issues.
