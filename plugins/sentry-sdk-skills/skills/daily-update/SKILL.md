---
name: daily-update
description: Generate a formatted async daily standup message for the Sentry SDK team channel. Use when asked to create a "daily update", "async daily", "standup update", or "async team update".
---

# Async Daily Update

Generate a properly formatted async daily standup message for posting in the team Slack channel.

## Step 1: Gather Information

Ask the user for the following. Collect all answers before generating the message.

**Required:**
- What are you working on today? (list each task; ask if any have Linear issue links to include)

**Optional — ask about each, only include in output if the user has something to report:**
- Any blockers? (something preventing you from making progress)
- Anything you need from a specific teammate? (who and what)
- Any time OOO today? (if yes, ask for the time range and time zone)

## Step 2: Generate the Message

Format the message exactly as follows:

```
🔄 YYYY-MM-DD

- [task one]
- [task two — add link if provided]
- **Blocker:** [description]
- **Needs:** [what you need — @mention the person directly]
- **OOO:** [time range with time zone, e.g., 14:00–17:00 CET]
```

### Rules

- Always start with the 🔄 emoji — this makes the update searchable in Slack
- Use today's date in `YYYY-MM-DD` format
- Leave a blank line between the date and the first bullet
- Each task is a separate bullet point
- If the user provides a Linear issue link, inline it after the task description: `Task description [SDK-123](https://linear.app/...)`
- Only include **Blocker**, **Needs**, and **OOO** lines if the user has something to report — omit them entirely otherwise
- For **Needs**, mention the specific person by name or @handle
- For **OOO**, always include the time zone (e.g., CET, EST, PST)

## Step 3: Output

Display the final message as a plain code block so the user can copy and paste it directly into Slack.

After the code block, remind the user: post this in your `#team-sdk-*` channel.

## Examples

**Minimal (no optional fields):**
```
🔄 2024-02-16

- Finishing OTLP exporter implementation [SDK-234](https://linear.app/team/issue/SDK-234)
- Code review for Marco's span processor refactor [SDK-245](https://linear.app/team/issue/SDK-245)
```

**With optional fields:**
```
🔄 2024-02-16

- Wrapping up bugfixes in profiling integration and preparing release notes
- Shifting focus to client reports implementation after bugfixes merge
- **Blocker:** Need test credentials for staging environment to verify the span ingestion fix
- **Needs:** API design review on error handling from @Sarah
- **OOO:** 14:00–17:00 CET
```
