# How-To Guide: SDK Feature Alignment

Step-by-step guide for using sdk-feature-status and related skills to implement features across Sentry SDKs.

## Prerequisites

**Required:**
- Claude Code CLI installed
- `gh` CLI authenticated (`gh auth login`)
- sentry-sdk-skills plugin installed
- sentry-skills plugin installed

**Verify:**
```bash
claude --version
gh auth status
claude plugin list | grep sentry
```

## Step-by-Step Workflow

### Step 1: Check Feature Status Across All SDKs

```bash
# From any directory (will create .sdk-align/ here)
/sdk-feature-status https://github.com/getsentry/sentry-docs/pull/12345
```

**What happens:**
1. Fetches develop doc from GitHub
2. Identifies reference implementation (or asks you)
3. Checks all 17 SDKs for:
   - PRs mentioning the develop doc
   - Commits with feature keywords
   - Code patterns from reference implementation
4. Creates `.sdk-align/` directory with:
   - `context.json` - Feature details, develop doc, reference impl
   - `status-report.json` - Implementation status per SDK

**Output example:**
```
✅ Implemented: javascript, python
⚠️  Needs Review: go (open PR)
❌ Not Implemented: ruby, java, php, dotnet, rust, elixir
🚫 Not Applicable: cocoa, android (mobile-only feature)

Summary: 2 implemented, 1 in review, 7 need implementation
```

**Time:** 2-5 minutes depending on API rate limits

---

### Step 2: Review Context Files

**Check what was found:**
```bash
cat .sdk-align/context.json
```

**Example context.json:**
```json
{
  "feature": "client-reports",
  "develop_doc": {
    "url": "https://github.com/getsentry/sentry-docs/pull/12345",
    "type": "pr"
  },
  "reference_implementation": {
    "sdk": "javascript",
    "pr_url": "https://github.com/getsentry/sentry-javascript/pull/6789",
    "pr_number": 6789
  },
  "alignment_scope": {
    "domain": ["backend", "frontend"],
    "sdks": ["python", "javascript", "ruby", "php", "go", "java", "dotnet"]
  },
  "linear_issue": "",
  "timestamp": "2024-01-22T10:30:00Z"
}
```

**Check status report:**
```bash
cat .sdk-align/status-report.json | jq '.sdks[] | select(.status == "not_implemented")'
```

---

### Step 3: Create Linear Issues for Tracking (Optional)

If you want Linear issues for each SDK team:

**Option A: Create issues from context manually**

```bash
"Create a Linear issue for the Python team based on .sdk-align/context.json

Title: Implement Client Reports in sentry-python
Description:
- Feature: Client Reports
- Develop doc: <url from context.json>
- Reference: JavaScript implementation <pr_url from context.json>
- Status: Not implemented (from status-report.json)

Team: Python
Labels: feature, sdk-alignment"
```

**Option B: Use linear-initiative skill**

```bash
/linear-initiative

# Prompts for:
# - Initiative name (or link to existing)
# - Select SDK teams
# Creates project per team linked to initiative
```

**What it does:**
- Creates Linear project per SDK team
- Links to parent initiative
- Can reference context.json for details

---

### Step 4: Implement Feature in First SDK

Start with 1-2 SDKs to validate approach.

#### For Simple Features (Direct Implementation)

```bash
# Navigate to SDK repository
cd ~/code/sentry-python

# Ask Claude to implement
"Implement the feature from .sdk-align/context.json

Requirements:
- Read the develop doc and reference implementation
- Follow Python SDK conventions
- Add comprehensive tests
- Add docstrings

Let me review before committing."
```

**Claude will:**
1. Read context.json
2. Fetch develop doc and reference PR
3. Generate Python implementation
4. Show you the changes

**Review the changes:**
```bash
git diff --stat
git diff
```

#### For Complex Features (Use Plan Mode)

```bash
cd ~/code/sentry-go

/plan
"Implement client reports from .sdk-align/context.json

This is complex - design the approach first:
1. Study the Go SDK architecture
2. Analyze how similar features are implemented
3. Design idiomatic Go implementation
4. Identify all files that need changes
5. Plan test strategy

Get my approval before implementing."
```

**Claude will:**
1. Explore Go SDK codebase
2. Study existing patterns
3. Design implementation approach
4. Show plan for approval
5. After approval → implement

---

### Step 5: Use Sentry Skills for Commit

After implementation and testing:

```bash
# Stage changes
git add .

# Use sentry-skills:commit for proper formatting
/commit
```

**What /commit does:**
1. Analyzes your changes
2. Prompts for:
   - Type: `feat` (for new feature)
   - Scope: `python` (SDK name)
   - Description: Auto-generated or custom
3. Creates commit with Sentry conventions:
```
feat(python): Add client reports

Implements client reports feature for sentry-python SDK following
the specification from develop docs.

Based on reference implementation in sentry-javascript.

- Add ClientReportManager class
- Integrate with Transport layer
- Add configuration options
- Add comprehensive tests

Refs: https://github.com/getsentry/sentry-docs/pull/12345

Co-Authored-By: Claude <noreply@anthropic.com>
```

**Manual alternative if /commit not available:**
```bash
git commit -m "feat(python): Add client reports

Implements client reports feature.

Refs: https://github.com/getsentry/sentry-docs/pull/12345

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Step 6: Push and Create Pull Request

```bash
# Push branch
git push -u origin feat/client-reports

# Use sentry-skills:create-pr for proper formatting
/create-pr
```

**What /create-pr does:**
1. Checks you're on pushed branch with commits
2. Analyzes commits and diff
3. Checks for PR template
4. Prompts for additional context
5. Creates PR with Sentry conventions

**When prompted, provide:**
- Develop doc URL (from context.json): `https://github.com/getsentry/sentry-docs/pull/12345`
- Reference PR (from context.json): `https://github.com/getsentry/sentry-javascript/pull/6789`
- Linear issue (if created): `SDK-123`

**Resulting PR:**
```markdown
# Pull Request: Add client reports to sentry-python

## Summary

Implements client reports feature following the specification from develop docs.

## Changes

- Add ClientReportManager class to handle report aggregation
- Integrate with Transport layer for report sending
- Add configuration options: `send_client_reports` (default: true)
- Add comprehensive unit and integration tests

## References

- Develop doc: https://github.com/getsentry/sentry-docs/pull/12345
- Reference implementation: https://github.com/getsentry/sentry-javascript/pull/6789
- Linear issue: SDK-123

## Testing

- Unit tests: `pytest tests/test_client_reports.py`
- Integration test: `pytest tests/integrations/test_client_reports_integration.py`
- Manual testing with sample app

## Implementation Notes

Python implementation uses context manager pattern for report collection,
which is more idiomatic than the JavaScript callback approach.
```

**Manual alternative if /create-pr not available:**
```bash
gh pr create --title "feat(python): Add client reports" \
  --body "See commit message for details.

Refs:
- Develop doc: https://github.com/getsentry/sentry-docs/pull/12345
- Reference: https://github.com/getsentry/sentry-javascript/pull/6789"
```

---

### Step 7: Update Linear Issue with PR Link

**Option A: Manual**
```bash
# Copy PR URL, paste into Linear issue
# Update status to "In Review"
```

**Option B: Programmatic** (if Linear API access configured)
```bash
"Update Linear issue SDK-123 with PR link: <pr-url>
Change status to In Review"
```

---

### Step 8: Repeat for Other SDKs

```bash
cd ~/code/sentry-go

# Reference previous work
"Implement client reports from .sdk-align/context.json

I just implemented this in Python: ~/code/sentry-python/sentry/client_reports.py

Adapt the approach for Go:
1. Use /plan to design Go-idiomatic implementation
2. Follow Go SDK conventions
3. Ensure proper error handling and concurrency patterns"
```

**Tips for subsequent implementations:**
- Reference what you learned from first SDK
- Adapt patterns to target language idioms
- Note differences in status-report.json

---

## Advanced Usage

### Batch Implementation with Context Awareness

```bash
"Read .sdk-align/status-report.json

Implement client reports for all SDKs marked as 'not_implemented':
- Start with Python (I have it checked out at ~/code/sentry-python)
- Then Go (~/code/sentry-go)
- Then Ruby (~/code/sentry-ruby)

For each SDK:
1. Use /plan if complex
2. Generate idiomatic implementation
3. Show me the diff
4. I'll approve, then we commit with /commit
5. Create PR with /create-pr

Pause between each SDK so I can test."
```

### Update Context Manually

Edit `.sdk-align/context.json` to add information:

```bash
# Add Linear issue after creating it
jq '.linear_issue = "https://linear.app/sentry/issue/SDK-123"' \
  .sdk-align/context.json > .tmp && mv .tmp .sdk-align/context.json

# Add custom notes
jq '.notes = "Complex feature - use plan mode for each SDK"' \
  .sdk-align/context.json > .tmp && mv .tmp .sdk-align/context.json
```

### Query Status Programmatically

```bash
# List SDKs needing implementation
jq -r '.sdks[] | select(.status == "not_implemented") | .name' \
  .sdk-align/status-report.json

# Count by status
jq '.summary' .sdk-align/status-report.json

# Find SDKs with open PRs
jq -r '.sdks[] | select(.status == "needs_review") | "\(.name): \(.pr_url)"' \
  .sdk-align/status-report.json
```

---

## Common Questions

**Q: Do I need to run all skills from the same directory?**
A: Only `/sdk-feature-status` needs a specific directory (it creates `.sdk-align/` there). After that, cd to each SDK repo as needed.

**Q: Can I edit the context files manually?**
A: Yes, they're just JSON files. Edit them to add notes, Linear issues, or corrections.

**Q: What if I lose the context files?**
A: Run `/sdk-feature-status` again. Or work without them by pasting the develop doc URL in your prompts.

**Q: Can I use this for non-SDK work?**
A: The status checker is SDK-specific, but the workflow (plan mode + context files) applies to any coding task.

## Troubleshooting

### "gh: command not found"

```bash
# Install GitHub CLI
brew install gh  # macOS
# or download from: https://cli.github.com/

# Authenticate
gh auth login
```

### "Permission denied" accessing repositories

```bash
# Check auth status
gh auth status

# Re-authenticate
gh auth refresh -h github.com -s repo
```

### Context files not found

```bash
# Make sure you're in the same directory where you ran /sdk-feature-status
pwd
ls -la .sdk-align/

# If lost, just run status check again
/sdk-feature-status <develop-doc-url>
```

### "/commit or /create-pr not found"

```bash
# Install sentry-skills plugin
claude plugin install sentry-skills

# Verify
claude plugin list | grep sentry-skills
```

### Claude doesn't read context files

Be explicit in your prompt:
```bash
# Instead of: "Implement the feature"
# Say: "Read .sdk-align/context.json and implement the feature described there"
```

---

## Best Practices

### 1. Test with 1-2 SDKs First
Don't implement across all SDKs immediately. Validate your approach:
- Implement in reference SDK (usually JavaScript or Python)
- Implement in one other SDK (different language paradigm)
- Review and test both
- Then expand to remaining SDKs

### 2. Use Plan Mode for Complex Features
If the feature:
- Touches many files
- Involves architectural decisions
- You're unfamiliar with the SDK
- Has multiple implementation approaches

→ Use `/plan` first

### 3. Commit and PR After Each SDK
Don't batch commits. After each SDK:
- Commit with `/commit`
- Create PR with `/create-pr`
- Get review
- Fix issues
- Then move to next SDK

### 4. Reference Previous Implementations

```bash
"I implemented this in Python at ~/code/sentry-python/sentry/client_reports.py

Now implement for Ruby, but:
- Use Ruby idioms (blocks instead of callbacks)
- Follow Ruby SDK patterns
- Adapt the test strategy"
```

### 5. Keep Context Updated

Update `.sdk-align/` files as you go:
- Add Linear issues
- Note implementation decisions
- Track deviations from spec
- Add lessons learned

---

## Workflow Summary

```
1. /sdk-feature-status <develop-doc-url>
   └─> Creates .sdk-align/context.json + status-report.json

2. (Optional) Create Linear issues
   └─> /linear-initiative or manually

3. For each SDK:
   ├─> cd ~/code/sentry-<sdk>
   ├─> (Complex) /plan → approve → implement
   │   └─ or ─
   ├─> (Simple) Direct prompt with context.json reference
   ├─> Review changes
   ├─> /commit (sentry-skills:commit)
   ├─> git push
   ├─> /create-pr (sentry-skills:create-pr)
   │   └─> Include develop doc + reference PR links
   └─> Update Linear issue with PR link

4. Iterate across remaining SDKs
   └─> Reference previous implementations
```

---

## Example: Complete Session

```bash
# Start fresh
cd ~/projects/sdk-alignment

# 1. Check status
/sdk-feature-status https://github.com/getsentry/sentry-docs/pull/12345

# Output shows: python, go, ruby need implementation

# 2. Start with Python
cd ~/code/sentry-python

# 3. Implement
/plan
"Implement client reports from .sdk-align/context.json
Design approach first - this is complex"

# Review plan → approve

# 4. Test
pytest tests/test_client_reports.py -v

# 5. Commit
git add .
/commit
# Choose: feat, python, auto-description

# 6. Push and PR
git push -u origin feat/client-reports
/create-pr
# Provide: develop doc URL, reference PR URL

# 7. Update Linear
"Update Linear issue SDK-123 with PR: <url>"

# 8. Move to Go
cd ~/code/sentry-go

# 9. Repeat with context
"Implement client reports from .sdk-align/context.json
Reference Python implementation: ~/code/sentry-python/sentry/client_reports.py
Use /plan to design Go approach"

# Continue...
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Check status | `/sdk-feature-status <url>` |
| View context | `cat .sdk-align/context.json` |
| View status | `cat .sdk-align/status-report.json` |
| Plan implementation | `/plan` then prompt |
| Direct implementation | Prompt with context.json reference |
| Commit changes | `/commit` |
| Create PR | `/create-pr` |
| Create Linear tracking | `/linear-initiative` |
| Query SDKs needing work | `jq -r '.sdks[] \| select(.status == "not_implemented") \| .name' .sdk-align/status-report.json` |

---

## Further Reading

- [README.md](./README.md) - Overview and installation
- [Agent Skills Spec](https://agentskills.io/specification) - Skill format
- [Sentry Engineering](https://develop.sentry.dev/engineering-practices/) - Conventions
