---
name: kubernetes
description: >-
  List Kubernetes pods in a namespace formatted as a markdown table and debug
  pods in CrashLoopBackOff using kube_mcp tools or kubectl commands. Use when
  the user asks to list pods, check namespace status, or debug crash loops.
---

# Kubernetes Operations and Troubleshooting

This skill covers listing Kubernetes pods in namespaces and diagnosing pods stuck in a `CrashLoopBackOff` state using both MCP tools and command-line fallbacks.

## Workflow

Copy and track progress:

```
- [ ] 1. Identify namespace and targets
- [ ] 2. List pods in target namespace (format as table)
- [ ] 3. Identify pods in CrashLoopBackOff / Error states
- [ ] 4. Diagnose crash loop (inspect YAML, check previous logs, check events)
- [ ] 5. Summarize findings and propose resolution
```

---

## Part 1: List Pods in Namespace

When asked to list pods, always retrieve their details and format them into a clean Markdown table.

### 1. Retrieve Pods using MCP (Preferred)

If `kube_mcp` tools are available, use:

```json
kube_mcp-pods_list_in_namespace
  namespace: "<namespace>"
```

### 2. Retrieve Pods using Command Line (Fallback)

If the MCP tool is unavailable or fails, run:

```bash
kubectl get pods -n <namespace> -o wide
```

### 3. Formatting the Output

Always format the result as a Markdown table. Do not output raw JSON or unformatted command text. Use the following columns:

| Pod Name | Ready | Status | Restarts | Age | IP | Node |
|----------|-------|--------|----------|-----|----|------|
| `my-pod-abc-123` | 0/1 | CrashLoopBackOff | 12 | 2h | 10.244.1.15 | worker-node-1 |
| `ok-pod-xyz-987` | 1/1 | Running | 0 | 5d | 10.244.2.30 | worker-node-2 |

---

## Part 2: Debug CrashLoopBackOff

If a pod is crashing repeatedly (`CrashLoopBackOff` status or high restart counts with `Error` status), follow this debugging pipeline:

### Step 1: Inspect Pod Metadata & Container Status

Get detailed pod specifications and state transition reasons.

**MCP Tool:**
```json
kube_mcp-pods_get
  name: "<pod-name>"
  namespace: "<namespace>"
```

**Fallback:**
```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

**What to look for:**
- Under `status.containerStatuses[]`:
  - `state.waiting.reason`: Will typically be `CrashLoopBackOff`.
  - `state.waiting.message`: May contain details on the crash loop delay.
  - `lastState.terminated.exitCode`: The exit status of the crashed container.
  - `lastState.terminated.reason`: e.g., `OOMKilled`, `Error`, `ContainerCannotRun`.
  - `lastState.terminated.message`: Specific error from the container engine (if any).

### Step 2: Retrieve Container Logs

To understand why the application crashed, you must read the logs.
**CRITICAL**: Because the container is crashlooping, the *current* running container might have just started and have empty logs. You **MUST** retrieve the logs of the **previous terminated container** to see the actual error/exception traceback.

**MCP Tool:**
```json
kube_mcp-pods_log
  name: "<pod-name>"
  namespace: "<namespace>"
  container: "<container-name>"  # specify if multi-container
  previous: true                  # IMPORTANT: gets the crashed instance's logs
  tail: 100
```

**Fallback:**
```bash
kubectl logs <pod-name> -n <namespace> -c <container-name> --previous --tail=100
```

*Note: If `--previous` fails with "previous terminated container not found", check the current logs instead:*
```bash
kubectl logs <pod-name> -n <namespace> -c <container-name> --tail=100
```

### Step 3: Fetch Related Events

Check for cluster-level issues (like failing probes or volume mounts) that could trigger restarts.

**MCP Tool:**
```json
kube_mcp-events_list
  namespace: "<namespace>"
```

**Fallback:**
```bash
kubectl get events -n <namespace> --sort-by='.metadata.creationTimestamp'
```

**What to look for:**
- `Warning BackOff`: Confirms crashlooping.
- `Warning Unhealthy`: Liveness or Readiness probe failed. If Liveness probe fails, Kubelet kills and restarts the container.
- `Warning FailedMount`: Problem attaching volumes.
- `Warning FailedScheduling`: Cluster scheduling issues.

---

## Part 3: Common Exit Codes & Diagnosis

| Exit Code | Meaning / Reason | Common Fixes |
|-----------|------------------|--------------|
| **137** | `OOMKilled` (Out of Memory) | Increase resource memory limits in the deployment spec. Check for memory leaks. |
| **1** | Application Crash (uncaught exception) | Check previous container logs for traceback/stack trace. Check database URL, missing env vars, or syntax errors. |
| **127** | Command / Entrypoint not found | Check if Dockerfile CMD/ENTRYPOINT points to a file that does not exist or isn't in `$PATH`. |
| **139** | Segmentation fault | Native library compatibility issue, or library version mismatch. |
| **0** | Completed (normal exit) | The pod completed its task (common for Jobs). If it was supposed to be a long-running service, check if the app's entrypoint exited prematurely. |

---

## Part 4: Diagnosis Report Template

After completing the diagnosis, present the results to the user using this format:

```markdown
### Kubernetes CrashLoop Diagnosis: <pod-name>

- **Namespace:** <namespace>
- **Container:** <container-name>
- **Status:** CrashLoopBackOff (Restarts: <count>)
- **Termination Details:** Exit Code <exit-code>, Reason: <reason>

#### Error Logs Excerpt (Previous Container)
```
<insert 10-30 lines of relevant traceback or error logs>
```

#### Relevant Events
```
<insert events or warnings, e.g. Liveness probe failed or Warning BackOff>
```

#### Root Cause Analysis
<One or two sentences explaining exactly why the container crashed.>

#### Recommended Fix
1. <Actionable resolution step, e.g., increase memory limit to 512Mi>
2. <Additional resolution steps, e.g., correct database credentials in Secret>
```

## Rules

- Always format listed pods as a Markdown table.
- Never guess the reason for a crash loop. Always check the `--previous` container logs.
- Distinguish between a pod that completed its run successfully (`Exit Code 0`) and an actual failure.
