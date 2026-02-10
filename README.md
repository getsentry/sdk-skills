# Sentry SDK Skills

Agent skills for managing feature development across Sentry SDKs, following the [Agent Skills](https://agentskills.io) open format.

## What This Does

Automates checking implementation status across all 17 Sentry SDKs, then helps you implement features efficiently using Claude Code's natural workflows.

## Installation

### Claude Code

```bash
git clone git@github.com:getsentry/sdk-skills.git
claude plugin install ~/sdk-skills
```

Restart Claude Code after installation.

### Updating

```bash
claude plugin update sentry-sdk-skills
```

Or use `/plugin` for the interactive plugin manager.

## Skills

| Skill | Purpose |
|-------|---------|
| `sdk-feature-status` | Check which of the 17 SDKs have implemented a feature. Searches GitHub across all repos, analyzes PRs/commits/code, creates status report. |
| `linear-initiative` | Create Linear projects for SDK teams to track feature rollout. |

## Quick Start

### 1. Check Status

```bash
/sdk-feature-status https://github.com/getsentry/sentry-docs/pull/12345
```

Creates:
- `.sdk-align/status-report.json` - Which SDKs have/need the feature
- `.sdk-align/context.json` - Feature details and reference implementation

### 2. Implement Features

**Simple feature:**
```bash
cd ~/code/sentry-python
"Implement client reports from .sdk-align/context.json based on the JS reference"
/commit
/create-pr
```

**Complex feature:**
```bash
cd ~/code/sentry-go
/plan
"Implement client reports from .sdk-align/context.json - design the approach first"
# Review plan, then implement
```

**Batch implementation:**
```bash
"Look at .sdk-align/status-report.json and implement for SDKs that need it"
```

**📖 See [HOW-TO.md](./HOW-TO.md) for complete step-by-step guide**

## How It Works

1. **status skill** automates the tedious work (checking 17 repos, analyzing PRs, aggregating results)
2. **context files** store feature details you can reference in prompts
3. **Claude Code** + **plan mode** do the implementation work naturally
4. **sentry-skills** handles commit formatting and PR creation

This gives you automation where it matters (status checking) while keeping implementation flexible.

## Key Concepts

### Context Files

Reference in your prompts:
```bash
"Read .sdk-align/context.json and implement in Ruby"
"Which SDKs in .sdk-align/status-report.json need this?"
```

### Plan Mode

Use `/plan` for complex features:
```bash
/plan
"Implement client reports - design the approach first"
```

Plan mode:
- Explores the codebase
- Studies existing patterns
- Designs implementation strategy
- Gets your approval before coding

### Iterative Workflow

Start with 1-2 SDKs to validate your approach:
```bash
/sdk-feature-status <url>  # Get the landscape
# Implement in Python (reference)
# Implement in Go (test another language)
# Expand to remaining SDKs
```

## Dependencies

Integrates with [sentry-skills](https://github.com/getsentry/skills) plugin:
- `/commit` - Formats commits following Sentry conventions
- `/create-pr` - Creates PRs with proper Sentry formatting

Install sentry-skills alongside this plugin.

## Repository Structure

```
sdk-skills/
├── plugins/sentry-sdk-skills/
│   ├── .claude-plugin/plugin.json
│   ├── skills/
│   │   ├── feature-status/      # Status checker
│   │   └── linear-initiative/   # Linear integration
│   └── references/
│       └── sdk-mappings.md      # SDK to repo mapping
├── HOW-TO.md                    # Step-by-step guide
└── README.md                    # This file
```

## Why This Approach?

**What we automated:**
- Checking 17 SDK repositories (would take 30+ minutes manually)
- GitHub API searches for PRs, commits, code patterns
- Aggregating results into actionable report

**What we didn't automate:**
- Code generation (Claude Code + prompts work better)
- PR creation (delegated to sentry-skills:create-pr)
- Repository management (use git normally)

Result: Automation where it provides real value, flexibility everywhere else.

## Documentation

**[HOW-TO.md](./HOW-TO.md)** - Complete step-by-step guide:
- Check status across all SDKs
- Use context files in prompts
- Implement with plan mode
- Commit and create PRs
- Troubleshooting and best practices
- Full example session

## References

- [Agent Skills Specification](https://agentskills.io/specification)
- [Sentry Engineering Practices](https://develop.sentry.dev/engineering-practices/)
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)

## License

Apache-2.0
