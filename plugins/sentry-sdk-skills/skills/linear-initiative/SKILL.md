---
name: linear-initiative
description: Creates Linear projects for SDK repositories based on a Linear initiative. Use when rolling out a feature across multiple SDKs, creating SDK projects from initiative, or setting up cross-SDK alignment work.
allowed-tools: Read mcp__linear-server__query_data mcp__linear-server__list_teams mcp__linear-server__list_projects mcp__linear-server__create_project mcp__linear-server__get_project mcp__linear-server__update_project mcp__linear-server__create_issue mcp__linear-server__update_issue AskUserQuestion
compatibility: Requires the Linear MCP server to be configured
---

# Linear Project Creator

Uses the content of a Linear initiative and creates projects for selected SDKs.

## Requirements

This skill requires the [Linear MCP server](https://github.com/linear/linear-mcp) to be configured.

## SDK Teams

Only teams with Enabled=Yes are included when creating projects.

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

## Project Naming

**Naming rules**:
1. Remove any text in parentheses or brackets from the initiative name
2. Add suffix in square brackets

**Format**: `[Clean Initiative Name] [[Suffix]]`

Example: Initiative "SDK Handling HTTP 413 (Content Too Large)" → "SDK Handling HTTP 413 [Python]"

## Instructions

1. **Get the initiative identifier** from the user if not provided. This can be an initiative ID, name, or URL.

2. **Fetch the initiative details** using `mcp__linear-server__query_data`:
   ```
   Query: "Get initiative [identifier] with its name, description, targetDate, and any associated projects"
   ```

3. **Present initiative summary** to the user including:
   - Initiative name
   - Description/content
   - Target date (if set)
   - Any existing projects already linked

4. **Ask which SDK teams** should have projects created. Present options:
   - All SDK teams (all enabled teams from the table)
   - By category: JavaScript, Backend, Mobile, or Gaming (filter by Category column, only enabled)
   - Individual selection (let user specify specific teams)

5. **If user selects individual**, ask them to specify which teams from the table.

6. **Ask about project naming**:
   - Default: Use initiative name with team suffix (e.g., "Session Replay [Python]")
   - No suffix: Use initiative name only (same name for all projects)
   - Custom: Let user specify a custom naming pattern

7. **For each selected team**, create a project using `mcp__linear-server__create_project`:
   - **name**: Use the initiative name with suffix from the SDK Teams table (if suffixes enabled)
   - **team**: The team name
   - **description**: "Have a look at the [initiative](<initiative-url>) for a full description of this project."
   - **initiative**: Link to the source initiative
   - **state**: Set to "Backlog" (use status ID `d4127831-7550-41f2-918c-d7699c6c7b95`)
   - **targetDate**: If the initiative has a target date set, apply it to the project

8. **Report results** showing:
   - Successfully created projects with links
   - Any failures and reasons

9. **(Optional) Ask about creating issues** in each project:
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

10. **(Optional) Ask about creating GitHub-linked issues** in each Linear project:
    - After creating projects (and optionally issues in step 9), ask: "Do you also want to create issues that reference the GitHub SDK repositories?"
    - **Require explicit confirmation** before proceeding
    - If confirmed, create one issue per project using `mcp__linear-server__create_issue`:
      - **title**: Same as project name (e.g., "SDK Handling HTTP 413 [Python]")
      - **team**: The team name
      - **project**: Link to the created project
      - **description**: Include a link to the SDK repo (`https://github.com/getsentry/<repo-name>`) and a summary of the initiative requirements
    - Note: If issues were already created in step 9, skip this step — instead, update the existing issue descriptions using `mcp__linear-server__update_issue` to append the repo link.

## Example Usage

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

## Notes

- Always confirm the initiative details before creating projects
- Check for existing projects to avoid duplicates
- Projects are automatically linked to the initiative when the `initiative` parameter is provided
- If a project already exists for a team under this initiative, skip it and notify the user
- **GitHub Sync Warning**: Some teams have Linear-GitHub sync enabled. Issues created in Linear may automatically create corresponding GitHub issues. Always confirm with the user before creating issues.
