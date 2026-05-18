# Jenkins failure patterns (dev-sec-ops lab)

## Trivy / Harbor image scan

**Symptoms**

```text
No such image: harbor.tpinf.xyz/my-project/...
UNAUTHORIZED: project my-project not found
FATAL ... image scan error
```

**Cause:** In `jenkins/demo/Jenkinsfile`, `Build Container Image` tags `${IMAGE_NAME}` (`harbor.tpinf.xyz/lab/getting-started-todo-app:<build>`) but `Vulnerability Scan` scans a **different** hardcoded name:

```groovy
def imageName = "${REGISTRY_URL}/my-project/getting-started-todo-app:${env.BUILD_NUMBER}"
```

**Fix:** Use `env.IMAGE_NAME` or `${REGISTRY_URL}/${REGISTRY_PROJECT}/${APP_NAME}:${IMAGE_TAG}` in the Trivy stage.

---

## Harbor push

**Symptoms:** `docker push` denied, `unauthorized`, `unknown blob`

**Checks**

- Parameter `PUSH_TO_HARBOR` was `true`
- Jenkins credential `harbor-credentials` exists and matches Harbor admin
- Project `lab` exists in Harbor (not `my-project` unless created)
- Image was built successfully in prior stage

---

## SonarQube / SonarCloud

**Symptoms:** `sonar-scanner` exit non-zero, `401`, `403`, quality gate (if enabled)

**Checks**

- Jenkins credential id `sonar` (secret text token)
- `SONAR_PROJECT_KEY`, `SONAR_ORGANIZATION` in Jenkinsfile
- SonarCloud project exists and token has scan permission
- `withSonarQubeEnv('sonarqube')` matches Jenkins SonarQube server name

---

## DefectDojo import

**Symptoms:** `HTTP_STATUS:400/401/500`, connection refused

**Checks**

- Credential `defectdojo-api-token`
- `DEFECTDOJO_URL` — in-cluster: `http://defectdojo-django.defectdojo.svc.cluster.local`
- Engagement id `1` and product name exist
- Artifact `trivy-report.json` / `sonar-report.json` archived (may be empty if scan failed)

---

## Kubernetes agent pod

**Symptoms:** Long hang at `Pod [Pending]`, `ContainerCreating`, `ImagePullBackOff`

**Checks**

- Container images reachable from cluster (Docker Hub rate limits, etc.)
- `docker.sock` hostPath available on agent nodes
- Namespace `jenkins` resource quotas

---

## npm / Node stages

**Symptoms:** `npm ERR!`, `ENOENT`, test failures

**Checks**

- `client/` and `backend/` exist after clone
- `package-lock.json` vs `npm install` path in `installNodeDeps`
- Tests skipped when no `test` script (expected, not failure)

---

## MCP connectivity

**Symptoms:** Jenkins MCP tools error or timeout

**Checks**

- Pod `mcp-jenkins` running in `jenkins` namespace
- `mcp-jenkins-secret` password = Jenkins admin password (`admin.local` in lab)
- LiteLLM config: `http://mcp-jenkins.jenkins.svc.cluster.local:9887/sse`
- Cursor: `user-litellm` MCP server enabled
