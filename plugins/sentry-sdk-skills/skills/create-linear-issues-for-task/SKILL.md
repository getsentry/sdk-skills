---
name: create-linear-issues-for-task
description: Creates a parent Linear issue in a parent SDK team (Backend SDKs, JavaScript SDKs, Mobile SDKs, SDKs) with sub-task issues in the respective individual SDK team projects. Use when creating cross-SDK work items, rolling out features to SDK teams, or organizing SDK tasks with parent-child issue hierarchy.
allowed-tools: Read mcp__linear-server__query_data mcp__linear-server__list_teams mcp__linear-server__save_issue AskUserQuestion
compatibility: Requires the Linear MCP server to be configured
---

# Linear Parent Issue Creator

Creates a parent issue in a parent SDK team and sub-task issues in individual SDK teams, linked as children of the parent issue.

## Requirements

This skill requires the [Linear MCP server](https://github.com/linear/linear-mcp) to be configured.

## Parent Teams

These are the teams where the parent issue can be created:

| Parent Team | Category |
|-------------|----------|
| Backend SDKs | Backend |
| JavaScript SDKs | JavaScript |
| Mobile SDKs | Mobile |
| SDKs | General (also used for Gaming) |

## SDK Teams

Only teams with Enabled=Yes are included when creating sub-task issues.

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

When selecting SDK teams by category, use this mapping to determine which parent team hosts the parent issue:

| Category | Parent Team |
|----------|-------------|
| Backend | Backend SDKs |
| JavaScript | JavaScript SDKs |
| Mobile | Mobile SDKs |
| Gaming | SDKs |

## Issue Naming

**Naming rules**:
1. Remove any text in parentheses or brackets from the initiative name (if initiative-based)
2. Parent issue uses the clean name directly
3. Sub-task issues add a suffix in square brackets

**Parent issue**: `[Clean Name]`
**Sub-task format**: `[Clean Name] [[Suffix]]`

Example: "SDK Handling HTTP 413 (Content Too Large)"
- Parent issue: "SDK Handling HTTP 413"
- Sub-tasks: "SDK Handling HTTP 413 [Python]", "SDK Handling HTTP 413 [Go]", etc.

## Instructions

### Step 1: Determine Input Source

Ask the user how they want to provide the issue details:

- **From a Linear initiative**: Provide an initiative ID, name, or URL
- **Manual input**: Provide a title and description directly

### Step 2: Gather Issue Details

**If initiative-based:**

1. Fetch the initiative details using `mcp__linear-server__query_data`:
   ```
   Query: "Get initiative [identifier] with its name, description, targetDate, and any associated projects"
   ```
2. Present the initiative summary to the user:
   - Initiative name
   - Description/content
   - Target date (if set)
3. Use the initiative name (cleaned of parentheses/brackets) as the issue title
4. Use the initiative description as the issue description, prepended with a link: "Related initiative: [initiative-name](<initiative-url>)\n\n[description]"

**If manual input:**

1. Ask the user for:
   - **Title**: The issue title (will be used for parent and sub-tasks)
   - **Description**: The issue description/body

### Step 3: Select Parent Team

Ask the user which parent team should host the parent issue. Present the options from the Parent Teams table. If the user has already indicated a category (e.g., "create issues for all Backend SDKs"), infer the parent team from the Category to Parent Team Mapping.

### Step 4: Select SDK Teams for Sub-tasks

Ask the user which SDK teams should receive sub-task issues. Present options:

- **All enabled teams** under the selected parent team's category
- **By category**: JavaScript, Backend, Mobile, or Gaming (filter by Category column, only enabled teams)
- **Individual selection**: Let user specify specific teams from the SDK Teams table

If a team belongs to multiple categories (e.g., sentry-capacitor is both JavaScript and Mobile), include it if any of its categories match.

### Step 5: Confirm Before Creating

Present a summary to the user before creating anything:

- **Parent issue**: Title, parent team, description preview
- **Sub-task issues**: List of teams and their issue titles (with suffixes)
- **Total issues to create**: 1 parent + N sub-tasks

**Require explicit confirmation** before proceeding.

### Step 6: Create Parent Issue

Create the parent issue using `mcp__linear-server__save_issue`:
- **title**: The clean issue title (no suffix)
- **team**: The selected parent team name
- **description**: The issue description

Record the returned issue ID — this will be used as the `parentId` for all sub-tasks.

### Step 7: Create Sub-task Issues

For each selected SDK team, create a sub-task issue using `mcp__linear-server__save_issue`:
- **title**: Issue title with team suffix (e.g., "SDK Handling HTTP 413 [Python]")
- **team**: The individual SDK team name (e.g., "sentry-python")
- **description**: Same description as the parent issue
- **parentId**: The ID of the parent issue created in Step 6

### Step 8: Report Results

Present a summary of all created issues:
- Parent issue with link
- Each sub-task issue with link and team
- Any failures and reasons

## Example Usage

### Example 1: From initiative

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

### Example 2: Manual input

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

- Always confirm the full plan before creating any issues
- The parent issue must be created first to obtain its ID for linking sub-tasks
- Sub-task issues live in the individual SDK team's space, NOT in the parent team
- If issue creation fails for a sub-task, continue with remaining teams and report failures at the end
- **GitHub Sync Warning**: Some teams have Linear-GitHub sync enabled. Issues created in Linear may automatically create corresponding GitHub issues. Always warn the user about this before creating issues.
