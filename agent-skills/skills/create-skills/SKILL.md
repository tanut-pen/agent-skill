---
name: create-skills
description: Use this skill whenever the user wants to create a new Agent Skill (SKILL.md) for Claude. Triggers include: "create a skill", "make a skill", "build a skill", "write a SKILL.md", "new skill for agent", "custom skill", or any request to package Claude instructions into a reusable skill. Always use this skill when the user wants to define reusable behavior or domain expertise for Claude agents — even if they don't use the word "skill" explicitly.
---

# Create Skills

A skill for authoring new Agent Skills — reusable, filesystem-based capabilities that extend Claude with domain-specific expertise.

## What is a Skill?

A Skill is a directory containing a `SKILL.md` file (plus optional bundled resources) that Claude loads on-demand when relevant. Skills use **progressive disclosure**: only metadata is always loaded; the full instructions are fetched only when triggered.

```
my-skill/
├── SKILL.md          ← Required. YAML frontmatter + instructions
├── references/       ← Optional. Extra docs, schemas, API refs
│   └── guide.md
└── scripts/          ← Optional. Python/bash scripts Claude can run
    └── process.py
```

---

## Step 1 — Clarify Intent

Before writing anything, confirm:

1. **What should this skill enable Claude to do?**
2. **When should it trigger?** (user phrases, contexts, file types)
3. **What's the expected output?** (a file, inline text, code, etc.)
4. **Are there dependencies?** (Python packages, external APIs, specific tools)

Gather example inputs/outputs if available.

---

## Step 2 — Write SKILL.md

Every skill starts with YAML frontmatter:

```markdown
---
name: your-skill-name
description: What it does and when to use it. Be specific about triggers.
---

# Skill Title

## Overview
Brief summary of what this skill does.

## Workflow
Step-by-step instructions Claude should follow.

## Examples
Concrete input/output examples.
```

### Frontmatter rules

| Field | Rules |
|-------|-------|
| `name` | ≤64 chars, lowercase + hyphens only, no "anthropic"/"claude" |
| `description` | ≤1024 chars, no XML tags, must state WHAT and WHEN |

### Description writing tips

The description is the **primary trigger mechanism** — Claude decides whether to use a skill based on this text alone. Make it:

- **Specific**: name file types, user phrases, contexts
- **Pushy**: lean toward over-triggering rather than under-triggering
- **Dual-purpose**: explain both what it does AND when to use it

**Weak**: `"Helps with Excel files"`
**Strong**: `"Create and edit Excel spreadsheets (.xlsx). Use when the user mentions spreadsheets, wants to analyze tabular data, build reports, or work with any .xlsx/.csv file — even if they don't say 'Excel' explicitly."`

---

## Step 3 — Structure the Instructions

Keep `SKILL.md` body **under 500 lines**. Use progressive disclosure:

- **Put in SKILL.md**: core workflow, quick-start examples, pointers to other files
- **Put in `references/`**: lengthy API docs, schemas, edge-case guides
- **Put in `scripts/`**: deterministic operations (file transforms, validation)

Reference additional files clearly:

```markdown
For advanced form filling, see [references/forms.md](references/forms.md).
```

---

## Step 4 — Validate the Skill

Run a quick sanity check:

1. Does the `name` meet format requirements?
2. Is the `description` ≤1024 chars and trigger-specific?
3. Are instructions clear enough for Claude to follow without extra context?
4. If scripts are included, do they work standalone?

---

## Step 5 — Package and Install

### On claude.ai
Zip the skill folder and upload via **Settings → Features → Custom Skills**.

```bash
zip -r my-skill.skill my-skill/
```

### On Claude API
Upload via the `/v1/skills` endpoint with the required beta headers:
- `code-execution-2025-08-25`
- `skills-2025-10-02`
- `files-api-2025-04-14`

### On Claude Code
Place the folder in:
- `~/.claude/skills/` — personal, available in all projects
- `.claude/skills/` — project-local

---

## Minimal working example

```markdown
---
name: csv-summarizer
description: Summarize CSV files with statistics and key insights. Use when the user uploads a .csv file or asks to analyze tabular data, even if they don't say "CSV".
---

# CSV Summarizer

## Workflow
1. Read the CSV file with pandas
2. Compute row count, column names, data types, and null counts
3. For numeric columns, compute mean, min, max
4. Return a markdown summary table

## Example output
| Column | Type | Nulls | Min | Max | Mean |
|--------|------|-------|-----|-----|------|
| age    | int  | 0     | 18  | 65  | 34.2 |
```

---

## References

- [Agent Skills Overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Skills in the API](https://platform.claude.com/docs/en/build-with-claude/skills-guide)
- [agentskills.io](https://agentskills.io/home)