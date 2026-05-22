# Argo CD & manifest templates

Replace placeholders: `<name>`, `<namespace>`, `<service>`, `<chart>`, `<repo-url>`, `<values-file>`, `<path>`, `<release-name>`.

---

## Helm Application

Append to `argocd/app-list.yaml`:

```yaml
---
# ── <Service display name> ────────────────────────────────────────────────────
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <name>
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: devsecops-lab
spec:
  revisionHistoryLimit: 3
  project: default
  sources:
    - repoURL: <chart-repo-url>
      chart: <chart>
      targetRevision: "*"
      helm:
        releaseName: <release-name>   # omit if same as chart name
        valueFiles:
          - $values/<service>/<values-file>.yaml
    - repoURL: https://github.com/tanut-pen/dev-sec-ops.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

**Helm repo examples (already used in this repo)**

| Service | repoURL | chart |
|---------|---------|-------|
| Harbor | `https://helm.goharbor.io` | `harbor` |
| DefectDojo | `https://raw.githubusercontent.com/DefectDojo/django-DefectDojo/helm-charts` | `defectdojo` |
| Jenkins | `https://charts.jenkins.io` | `jenkins` |
| Kong | `https://charts.konghq.com` | `kong` |
| Grafana | `https://grafana.github.io/helm-charts` | `grafana` |
| Uptime Kuma | `https://helm.irsigler.cloud` | `uptime-kuma` |

---

## Kustomize Application

```yaml
---
# ── <Service display name> ────────────────────────────────────────────────────
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <name>
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: devsecops-lab
spec:
  revisionHistoryLimit: 3
  project: default
  source:
    repoURL: https://github.com/tanut-pen/dev-sec-ops.git
    targetRevision: HEAD
    path: <path>
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

---

## Static Ingress (Istio)

Add to `static/ingress.yaml` or new `static/<service>-ingress.yaml`:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <name>-ingress
  namespace: <namespace>
spec:
  ingressClassName: istio
  rules:
    - host: <service>.tpinf.xyz
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <k8s-service-name>
                port:
                  number: <port>
```

---

## Kustomize deployment + NodePort Service

`deployment.yaml` skeleton:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <name>
  namespace: <namespace>
  labels:
    app.kubernetes.io/name: <name>
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: <name>
  template:
    metadata:
      labels:
        app.kubernetes.io/name: <name>
    spec:
      containers:
        - name: <name>
          image: <image>:<tag>
          ports:
            - containerPort: <container-port>
          # env from secret.yaml:
          # env:
          #   - name: TOKEN
          #     valueFrom:
          #       secretKeyRef:
          #         name: <app>-secret
          #         key: TOKEN
---
apiVersion: v1
kind: Service
metadata:
  name: <name>
  namespace: <namespace>
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: <name>
  ports:
    - port: <port>
      targetPort: <container-port>
      nodePort: <30xxx>
```

---

## secret.yaml.example (committed)

```yaml
# Copy to secret.yaml (gitignored) and fill in real values.
apiVersion: v1
kind: Secret
metadata:
  name: <app>-secret
  namespace: <namespace>
type: Opaque
stringData:
  EXAMPLE_KEY: "changeme"
```

---

## install.sh (Kustomize)

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
kubectl apply -k "${SCRIPT_DIR}"
```
