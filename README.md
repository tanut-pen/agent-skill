# agent-skill

Agent skills for the **dev-sec-ops** lab. Each skill is a `SKILL.md` package that teaches Cursor (and other agents) how to run specialized workflows in this environment.

## Available skills

| Skill | Description |
|-------|-------------|
| **debug-jenkins** | Diagnose failed Jenkins builds using MCP tools, console logs, and pipeline context (`jenkins.tpinf.xyz`, `devsecops-demo-pipeline`, Trivy, Harbor, SonarQube). |
| **kubernetes** | List Kubernetes pods in a namespace formatted as a markdown table and debug pods in CrashLoopBackOff using kube_mcp tools or kubectl commands. |
| **new-service-deployment** | Scaffold a new service: Helm + Argo CD, or Kustomize + Argo CD, plus `static/` ingress and shared config for the dev-sec-ops repo. |

Skill sources live under [`agent-skills/skills/`](agent-skills/skills/).

## How to use this repository

Install skills with the [Skills CLI](https://skills.sh/) (`npx skills`). No global install is required.

### List skills in this repo

```bash
npx skills add tanut-pen/agent-skill --list
```

From a local clone:

```bash
npx skills add . --list
```

### Install all skills

```bash
npx skills add tanut-pen/agent-skill -y
```

Install globally (user-level, available in every project):

```bash
npx skills add tanut-pen/agent-skill -g -y
```

### Install one skill

```bash
npx skills add tanut-pen/agent-skill --skill debug-jenkins -y
```

```bash
npx skills add tanut-pen/agent-skill --skill new-service-deployment -y
```

Shorthand with `@`:

```bash
npx skills add tanut-pen/agent-skill@debug-jenkins -y
```

### Install from a local clone

```bash
git clone https://github.com/tanut-pen/agent-skill.git
cd agent-skill
npx skills add . -y
```

### Common options

| Flag | Purpose |
|------|---------|
| `-y`, `--yes` | Skip confirmation prompts |
| `-g`, `--global` | Install to user directory instead of the current project |
| `--skill <name>` | Install only the named skill |
| `--list` | Show available skills without installing |
| `-a <agent>` | Target a specific agent (e.g. `cursor`, `claude-code`) |

After install, skills are available to your agent (for Cursor, typically under `.cursor/skills/` or your configured skills path).

## Repository layout

```
agent-skills/
└── skills/
    ├── debug-jenkins/
    │   ├── SKILL.md
    │   └── failures.md
    └── new-service-deployment/
        ├── SKILL.md
        └── templates.md
```

## Learn more

- Browse and discover skills: [skills.sh](https://skills.sh/)
- Search the ecosystem: `npx skills find <query>`
- Check for updates: `npx skills check` / `npx skills update`
