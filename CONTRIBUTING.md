# Contributing to Sentry SDK Skills

Thank you for your interest in contributing to Sentry SDK Skills! This guide will help you get started.

## Development Setup

1. Clone the repository:
   ```bash
   git clone git@github.com:getsentry/sdk-skills.git
   cd sdk-skills
   ```

2. Install the plugin locally:
   ```bash
   # Note: ./ (not .) - the slash indicates a local path vs remote marketplace name (owner/repo)
   claude plugin marketplace add ./
   claude plugin install sentry-sdk-skills
   ```

## Creating a New Skill

1. Create a new directory under `plugins/sentry-sdk-skills/skills/`:
   ```bash
   mkdir -p plugins/sentry-sdk-skills/skills/my-new-skill
   ```

2. Create `SKILL.md` with the required frontmatter:
   ```yaml
   ---
   name: my-new-skill
   description: Clear description with keywords for agent detection
   model: sonnet
   allowed-tools: Read Grep Glob Bash
   ---

   # My New Skill

   Detailed instructions for the agent...
   ```

3. Update `README.md` to add your skill to the Available Skills table

4. Test the skill locally with Claude Code

## Skill Guidelines

### Frontmatter Requirements

- `name`: 1-64 characters, lowercase alphanumeric with hyphens only
- `description`: Up to 1024 characters, include trigger keywords that help agents identify when this skill is relevant
- `model` (optional): `sonnet`, `opus`, or `haiku`
- `allowed-tools` (optional): Space-delimited list of tools

### Content Guidelines

- Keep SKILL.md under 500 lines
- Split longer content into separate reference files
- Use clear, actionable language for agent instructions
- Include examples showing expected input/output
- Document edge cases and error handling
- Follow the [Agent Skills specification](https://agentskills.io/specification)

### Code Style

- Use markdown formatting consistently
- Include code blocks with language identifiers
- Use YAML frontmatter for metadata
- Keep instructions concise but complete

## Testing

Test your skill by:
1. Installing it locally in Claude Code
2. Invoking it with `/skill-name` or letting it auto-trigger
3. Verifying it handles edge cases correctly
4. Checking that error messages are clear

## Submitting Changes

1. Create a new branch:
   ```bash
   git checkout -b feature/my-new-skill
   ```

2. Make your changes and commit:
   ```bash
   git add .
   git commit -m "feat(skills): Add my-new-skill"
   ```

3. Push and create a pull request:
   ```bash
   git push origin feature/my-new-skill
   ```

4. In your PR description:
   - Explain what the skill does
   - Show example usage
   - List any dependencies or requirements

## Questions?

- Open an issue for bugs or feature requests
- Check existing issues before creating new ones
- Tag @getsentry/sdk-team for SDK-related questions

## License

By contributing, you agree that your contributions will be licensed under the Apache-2.0 License.
