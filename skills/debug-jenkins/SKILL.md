---
name: debug-jenkins
description: >-
  Diagnose failed Jenkins builds in the dev-sec-ops lab using Jenkins MCP tools,
  console logs, and pipeline context. Use when a Jenkins job fails, the user
  shares a jenkins.tpinf.xyz build or console URL, asks why a pipeline failed,
  or mentions devsecops-demo-pipeline, Trivy, Harbor, or SonarQube errors.
---

# Debug Jenkins Errors

## Lab context

| Item | Value |
|------|--------|
| Jenkins UI | https://jenkins.tpinf.xyz |
| MCP server | `mcp-jenkins` in namespace `jenkins` (lanbaoshen/mcp-jenkins) |
| MCP via Cursor | `user-litellm` tools prefixed `jenkins_mcp-` |
| Demo pipeline | `jenkins/demo/Jenkinsfile` → job `devsecops-demo-pipeline` |
| MCP credentials | `jenkins/mcpserver/secret.yaml` → must match `jenkins-values.yaml` admin password |

## Workflow

Copy and track progress:

```
- [ ] 1. Resolve job name + build number
- [ ] 2. Fetch build metadata (result, duration, URL)
- [ ] 3. Fetch console output (errors first, then tail)
- [ ] 4. Identify failing stage + root cause
- [ ] 5. Propose fix (Jenkinsfile, credentials, infra)
- [ ] 6. Report to user
```

### Step 1: Parse the target

From a Jenkins URL, extract `fullname` and `number`:

| URL pattern | fullname | number |
|-------------|----------|--------|
| `/job/devsecops-demo-pipeline/3/console` | `devsecops-demo-pipeline` | `3` |
| `/job/folder/job/my-job/5/console` | `folder/my-job` | `5` |

If only a job name is given, use `number: null` on console tool to get the **last** build.

### Step 2: MCP tools (preferred)

Read tool schemas under `mcps/user-litellm/tools/jenkins_mcp-*.json` before calling.

**Always call first:**

```
jenkins_mcp-get_build
  fullname: <job>
  number: <build>   # omit for last build
```

**Then console:**

```
jenkins_mcp-get_build_console_output
  fullname: <job>
  number: <build>
  pattern: "ERROR|FATAL|FAILURE|Exception|error:|failed|UNAUTHORIZED|denied|not found"
  limit: 60
```

If the failure is unclear, fetch the tail (no pattern, use `offset` on a second call or increase `limit`).

**Optional:**

| Need | Tool |
|------|------|
| Parameters used | `jenkins_mcp-get_build_parameters` |
| Test failures | `jenkins_mcp-get_build_test_report` |
| Compare prior build | `jenkins_mcp-get_build` with `number - 1` |
| Re-run | `jenkins_mcp-build_item` (confirm with user first) |
| Running builds | `jenkins_mcp-get_running_builds` |

### Step 3: Map console → stage

Match log markers to `jenkins/demo/Jenkinsfile` stages:

| Stage | Log / container clues |
|-------|------------------------|
| Clone Repository | `git`, `checkout` |
| Install Dependencies | `container('nodejs')`, `npm ci` |
| Run Tests | `npm test` |
| SonarQube Analysis | `sonar-scanner`, `SONAR_TOKEN` |
| Import SonarQube to DefectDojo | `import-scan`, `SONARQUBE IMPORT` |
| Build Container Image | `container('docker-cli')`, `docker build`, `IMAGE_NAME` |
| Vulnerability Scan | `container('trivy')`, `trivy image` |
| Push Image to Harbor | `docker login`, `docker push`, `PUSH_TO_HARBOR` |
| Import Scan to DefectDojo | `Trivy Scan`, `IMPORT_TO_DEFECTDOJO` |

### Step 4: Common failures in this repo

See [failures.md](failures.md) for patterns and fixes. Quick hits:

- **Trivy `No such image` / `project my-project not found`**: `Vulnerability Scan` uses hardcoded `my-project` but `environment` uses `REGISTRY_PROJECT = 'lab'` and `IMAGE_NAME` for the build — image name mismatch. Fix Jenkinsfile line ~232 to use `${IMAGE_NAME}` or `${REGISTRY_URL}/${REGISTRY_PROJECT}/...`.
- **Harbor UNAUTHORIZED**: Wrong project name, missing `harbor-credentials`, or image never pushed (`PUSH_TO_HARBOR` false).
- **SonarQube auth**: Check Jenkins credential id `sonar` and SonarCloud token.
- **DefectDojo import**: Check `defectdojo-api-token`, engagement id, pod reachability `defectdojo-django.defectdojo.svc`.
- **Agent pod stuck**: `ContainersNotReady`, image pull errors — check `jenkins_mcp-get_build_console_output` pod YAML section.
- **npm / tests**: Missing `package.json` scripts or network to registry.

Always read the actual `Jenkinsfile` in the repo before suggesting edits — jobs may differ from the demo.

### Step 5: Fallbacks if MCP fails

1. Confirm MCP pod: `kubectl get pods -n jenkins -l app.kubernetes.io/name=mcp-jenkins`
2. Check MCP logs: `kubectl logs -n jenkins deploy/mcp-jenkins --tail=50`
3. Verify secret matches Jenkins admin password (`jenkins/mcpserver/secret.yaml`)
4. Jenkins API (lab): `curl -s -u admin:<password> "https://jenkins.tpinf.xyz/job/<job>/<build>/consoleText" | tail -80`

Do not guess passwords; use repo docs or ask the user.

## Report template

```markdown
## Jenkins build failure

**Job:** <fullname> #<number>
**URL:** <build url>
**Result:** <SUCCESS|FAILURE|UNSTABLE|ABORTED>
**Duration:** <duration>

### Failing stage
<stage name>

### Error excerpt
```
<3–15 relevant log lines>
```

### Root cause
<one clear sentence>

### Recommended fix
1. <actionable step>
2. <optional step>

### Files to check
- `jenkins/demo/Jenkinsfile` (or job's actual script path)
- <credentials / harbor / sonar as needed>
```

## Rules

- Use MCP tools first; do not ask the user to paste the full console if MCP works.
- Quote real log lines; do not invent errors.
- Distinguish **symptom** (e.g. Trivy exit 1) from **root cause** (wrong image name in Jenkinsfile).
- Propose minimal Jenkinsfile/config fixes; do not refactor unrelated stages.
- For `build_item` or `stop_build`, confirm with the user before running.

## Additional resources

- Failure pattern reference: [failures.md](failures.md)
- Pipeline source: `jenkins/demo/Jenkinsfile`
- MCP deployment: `jenkins/mcpserver/`
