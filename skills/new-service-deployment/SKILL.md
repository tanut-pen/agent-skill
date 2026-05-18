---
name: new-service-deployment
description: >-
  Scaffold a new service in the dev-sec-ops lab: Helm chart + Argo CD app-list
  entry, or Kustomize manifests + app-list entry. Puts Ingress and shared static
  config under static/. Uses secret.yaml (gitignored) for Kustomize secrets.
  Use when adding a new tool, app, deployment, Helm release, or Argo CD
  Application to this repository.
---

# New Service Deployment

## Decision tree

```
New service?
├─ Official Helm chart exists (Harbor, DefectDojo, Jenkins, Kong, …)?
│  └─ Path A: Helm + dual-source Argo CD → argocd/app-list.yaml
└─ Simple app / no suitable chart / custom Deployment?
   └─ Path B: Kustomize + single-source Argo CD → argocd/app-list.yaml

Ingress / cluster-wide static config?
└─ Always Path C: static/ (Argo app static-manifest already syncs it)

Secrets (passwords, tokens)?
└─ Kustomize only: <service>/secret.yaml (gitignored), name file secret.yaml
```

Do **not** create a separate “helmlist” file — all Applications live in **`argocd/app-list.yaml`**. The root `app-of-apps` Application watches that file only.

Git repo URL for Argo CD sources: `https://github.com/tanut-pen/dev-sec-ops.git` (match existing entries in `app-list.yaml`).

---

## Path A — Helm-based service

### 1. Create service directory

```
<service>/
├── install.sh              # helm repo add + upgrade --install (local/manual)
├── <service>-values.yaml   # Helm values (committed)
└── readme.md               # only if user asks
```

**`install.sh` pattern** (from `harbor/install.sh`, `defectdojo/install.sh`):

```bash
#!/usr/bin/env bash
set -euo pipefail

helm repo add <repo-name> <chart-repo-url> --force-update
helm repo update

helm upgrade --install <release-name> <repo-name>/<chart> \
  --namespace <namespace> \
  --create-namespace \
  -f <service>-values.yaml
```

Add one-time bootstrap `--set` flags only in `install.sh`, not in Argo CD (see DefectDojo comment in `app-list.yaml`).

### 2. Values file

- Copy structure from a similar app (`harbor/harbor-values.yaml`, `defectdojo/defectdojo-values.yaml`).
- Set `ingress.enabled: false` when using **`static/`** Istio ingress instead of chart ingress.
- Prefer `NodePort` + fixed port for lab access (see `CLAUDE.md` port table; avoid collisions).

### 3. Add Argo CD Application to `argocd/app-list.yaml`

Append a `---`-separated block. Template in [templates.md](templates.md#helm-application).

Required fields:

- `metadata.name`: short name (e.g. `harbor`, `defectdojo`)
- `sources[0]`: public chart `repoURL`, `chart`, `targetRevision: "*"`
- `sources[0].helm.valueFiles`: `$values/<service>/<values-file>.yaml`
- `sources[0].helm.releaseName`: if install.sh uses non-default release name
- `sources[1]`: git repo with `ref: values`
- `destination.namespace`: matches Helm namespace
- `syncPolicy.automated`: `selfHeal: true`, `prune: true` (except Argo CD itself)
- `syncOptions`: `CreateNamespace=true`
- Label: `app.kubernetes.io/part-of: devsecops-lab`

### 4. Ingress

Add to **`static/ingress.yaml`** (or `static/<service>-ingress.yaml` like `harbor-ingress.yaml`):

- `ingressClassName: istio`
- `host: <service>.tpinf.xyz`
- `backend.service.name` / `port` must match what the **chart** creates (inspect values or `kubectl get svc -n <ns>` after first sync).

Do **not** put Ingress in the Helm values if the repo standard is central static ingress.

---

## Path B — Kustomize-based service

### 1. Create service directory

```
<service>/
├── kustomization.yaml
├── deployment.yaml       # Deployment + Service (same file or split)
├── secret.yaml           # gitignored — see Secrets section
├── secret.yaml.example   # optional committed template (no real secrets)
├── install.sh            # kubectl apply -k .
└── config/               # optional ConfigMap sources
```

**`kustomization.yaml` pattern** (from `litellm/kustomization.yaml`, `jenkins/mcpserver/`):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <namespace>

resources:
  - deployment.yaml
  # - secret.yaml   # uncomment locally after creating secret.yaml

labels:
  - pairs:
      app.kubernetes.io/part-of: devsecops-lab
      app.kubernetes.io/managed-by: kustomize
    includeSelectors: false
```

- **Do not** list `ingress.yaml` in resources — use `static/`.
- Use `configMapGenerator` for non-secret config (see `litellm/`).
- `generatorOptions.disableNameSuffixHash: true` when stable names matter.

### 2. Deployment / Service

- Follow `litellm/deployment.yaml` or `jenkins/mcpserver/deployment.yaml`.
- Lab: `Service` type `NodePort` with a free port (30433 LiteLLM, 30434 mcp-jenkins, 30001–30008 for charts).
- Set `namespace` in manifests or via kustomization `namespace:` field.

### 3. Add Argo CD Application to `argocd/app-list.yaml`

Single-source git path. Template in [templates.md](templates.md#kustomize-application).

```yaml
source:
  repoURL: https://github.com/tanut-pen/dev-sec-ops.git
  targetRevision: HEAD
  path: <service>   # or <service>/subdir
destination:
  namespace: <namespace>
```

### 4. Ingress → `static/`

Same as Path A step 4. Backend must match Kustomize `Service` name and port.

---

## Path C — Static manifests (`static/`)

Already synced by Application **`static-manifest`** (`path: static`, `directory.recurse: true`).

Add or extend:

| File | Use |
|------|-----|
| `static/ingress.yaml` | Shared Istio ingress rules (`*.tpinf.xyz`) |
| `static/harbor-ingress.yaml` | Complex multi-path ingress (example) |
| `static/kong-ingress.yaml` | Kong-specific routes |
| Other cluster config | TLS refs, shared middleware — keep here |

**Rules**

- One resource per YAML doc; separate with `---`.
- Set correct `metadata.namespace` on every object.
- Do **not** add a new Argo Application per ingress — only edit files under `static/`.
- Remind user to add DNS/hosts: `<host> → cluster IP`.

---

## Secrets

| Rule | Detail |
|------|--------|
| **Filename** | Always `secret.yaml` (not `*-secret.yaml`) |
| **Git** | Ignored by `.gitignore` pattern `secret*.yaml` |
| **Kustomize** | Comment out `secret.yaml` in `kustomization.yaml` by default; user uncomments locally |
| **Template** | Optional committed `secret.yaml.example` with placeholder keys |
| **Helm** | Prefer chart secrets or `install.sh --set`; do not commit real passwords in values YAML |

Example `secret.yaml` (Kustomize, local only):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <app>-secret
  namespace: <namespace>
type: Opaque
stringData:
  KEY: "value"
```

Reference in Deployment via `secretKeyRef` pointing at `metadata.name` above.

---

## Workflow checklist

Copy and track:

```
- [ ] Choose Path A (Helm) or Path B (Kustomize)
- [ ] Pick namespace and NodePort (no conflict with CLAUDE.md / existing services)
- [ ] Create <service>/ files + install.sh
- [ ] Add Ingress (and static-only config) under static/
- [ ] Append Application to argocd/app-list.yaml
- [ ] secret.yaml created locally if needed; kustomization commented until ready
- [ ] kubectl kustomize <path> OR helm template dry-run (validate)
- [ ] Commit only non-secret files; never commit secret.yaml
- [ ] Push → Argo CD app-of-apps picks up app-list.yaml
- [ ] Sync in UI or wait for automated sync
- [ ] Update CLAUDE.md service table if user wants lab docs
```

---

## Validation commands

```bash
# Kustomize
kubectl kustomize <service>/

# Helm (local)
helm template <release> <repo>/<chart> -f <service>/<values>.yaml -n <namespace>

# Argo (after push)
kubectl get applications -n argocd | grep <name>
```

---

## Do not

- Apply `app-list.yaml` directly (comment at top of file — use app-of-apps).
- Put Ingress in Kustomize/Helm service folders when repo standard is `static/`.
- Commit `secret.yaml` or real credentials in values files.
- Create duplicate Argo apps for `static/` — it is already managed.
- Use `prune: true` on stateful Helm apps without user confirmation if data loss risk exists.

---

## Reference

- Argo CD snippets: [templates.md](templates.md)
- Examples: `harbor/`, `defectdojo/` (Helm); `litellm/`, `jenkins/mcpserver/` (Kustomize)
- Static app: `argocd/app-list.yaml` → `static-manifest`
- Ports/credentials: `CLAUDE.md`
