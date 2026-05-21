# CLAUDE.md Context for agent-skill

This file provides context on how to run, verify, and maintain the agent skills repository.

## Commands

- **List local skills**: `npx --yes skills add . --list` (Verify skill parsing and CLI discovery)
- **Install skills locally**: `npx --yes skills add . -y`
- **Verify remote check**: `npx skills check`

## Repository Structure

```
agent-skills/
└── skills/
    ├── debug-jenkins/
    │   ├── SKILL.md
    │   └── failures.md
    ├── kubernetes/
    │   └── SKILL.md
    └── new-service-deployment/
        ├── SKILL.md
        └── templates.md
```

## Coding Guidelines & Standards

### Skill Structure
Each skill directory must contain a `SKILL.md` file. It can optionally contain supplementary helper markdown files (e.g. `failures.md`, `templates.md`).

### Frontmatter Metadata
The `SKILL.md` file must start with a YAML frontmatter block containing:
- `name`: Unique short identifier (kebab-case, e.g. `debug-jenkins`, `kubernetes`).
- `description`: 2-3 sentence description of the skill's purpose and triggering keywords (e.g., when the user mentions logs, pods, crash loops).

Example:
```yaml
---
name: kubernetes
description: >-
  List Kubernetes pods in a namespace formatted as a markdown table and debug
  pods in CrashLoopBackOff using kube_mcp tools or kubectl commands. Use when
  the user asks to list pods, check namespace status, or debug crash loops.
---
```

### Content Format
- **H1 Header**: Start the markdown body with a single `# <Title>`.
- **Workflow Checklist**: Include a copyable progress tracking checklist in a code block or checklist layout at the beginning:
  ```markdown
  - [ ] 1. Step one
  - [ ] 2. Step two
  ```
- **Tooling Hierarchy**:
  - Always prioritize MCP server tools (e.g., `jenkins_mcp-`, `kube_mcp-`) when they exist.
  - Provide fallback command-line options using native commands (e.g., `kubectl`, `curl`) if MCP tools are absent or fail.
- **Output Presentation**:
  - Always format query list results (like pod lists, build lists) into clean markdown tables rather than pasting raw JSON or command outputs.
  - Provide structured report templates (using markdown code blocks) for summarization workflows.
