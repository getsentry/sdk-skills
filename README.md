# Sentry SDK Skills

Agent skills for managing feature development across Sentry SDKs, following the [Agent Skills](https://agentskills.io) open format.

## Installation

### Claude Code (from local clone)

```bash
# Clone the repository
git clone git@github.com:getsentry/sdk-skills.git
cd sdk-skills

# Install the marketplace from the local clone
# Note: ./ (not .) - the slash indicates a local path vs remote marketplace name (owner/repo)
claude plugin marketplace add ./

# Install the plugin directly
claude plugin install sentry-sdk-skills
```

After installation, restart Claude Code. The skills will be automatically invoked when relevant to your task.

### Updating

```bash
# Update the marketplace index
claude plugin marketplace update

# Update the plugin
claude plugin update sentry-sdk-skills
```

Or use `/plugin` to open the interactive plugin manager.

### Other Agents

Copy the `skills/` directory to your agent's skills location, or reference the SKILL.md files directly according to your agent's documentation.

## Available Skills

| Skill | Description |
|-------|-------------|
| `daily-update` | Generate a formatted async daily standup message for the Sentry SDK team channel. Use when creating a "daily update", "async daily", or "standup update". |
| `linear-initiative` | Creates Linear projects for SDK teams from an initiative. Use when rolling out features across multiple SDKs. |
| `sdk-feature-implementation` | Implement a feature across SDK repos by spawning parallel agents — from GitHub issues or Linear context to draft PRs with CI verification. |


## Repository Structure

```
sdk-skills/
├── .claude-plugin/
│   └── marketplace.json      # Marketplace manifest
├── plugins/
│   └── sentry-sdk-skills/
│       ├── .claude-plugin/
│       │   └── plugin.json   # Plugin manifest
│       └── skills/
├── AGENTS.md                 # Agent-facing documentation
├── CLAUDE.md                 # Symlink to AGENTS.md
└── README.md                 # This file
```

## Creating New Skills

Skills follow the [Agent Skills specification](https://agentskills.io/specification). Each skill requires a `SKILL.md` file with YAML frontmatter.

### Skill Template

Create a new directory under `plugins/sentry-sdk-skills/skills/`:

```
plugins/sentry-sdk-skills/skills/my-skill/
└── SKILL.md
```

**SKILL.md format:**

```yaml
---
name: my-skill
description: A clear description of what this skill does and when to use it. Include keywords that help agents identify when this skill is relevant.
---

# My Skill Name

## Instructions

Step-by-step guidance for the agent.

## Examples

Concrete examples showing expected input/output.

## Guidelines

- Specific rules to follow
- Edge cases to handle
```

### Naming Conventions

- **name**: 1-64 characters, lowercase alphanumeric with hyphens only
- **description**: Up to 1024 characters, include trigger keywords
- Keep SKILL.md under 500 lines; split longer content into reference files

### Optional Fields

| Field | Description |
|-------|-------------|
| `license` | License name or path to license file |
| `compatibility` | Environment requirements (max 500 chars) |
| `model` | Override model for this skill (e.g., `sonnet`, `opus`, `haiku`) |
| `allowed-tools` | Space-delimited list of tools the skill can use |
| `metadata` | Arbitrary key-value pairs for additional properties |

```yaml
---
name: my-skill
description: What this skill does
license: Apache-2.0
model: sonnet
allowed-tools: Read Grep Glob
---
```

## References

- [Agent Skills Specification](https://agentskills.io/specification)
- [Sentry Engineering Practices](https://develop.sentry.dev/engineering-practices/)

## License

Apache-2.0
