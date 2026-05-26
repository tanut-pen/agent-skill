# Base Image Selection Guide

## Decision Tree

```
Does the runtime need a full OS shell?
├── No  → Use Distroless (preferred for security)
│         ├── Static binary (Go, Rust musl)?  → distroless/static-debian12   (~2 MB)
│         ├── Needs glibc only?               → distroless/base-debian12      (~20 MB)
│         ├── Node.js app?                    → distroless/nodejs20-debian12  (~100 MB)
│         └── JVM app?                        → distroless/java21-debian12    (~220 MB)
│
└── Yes → Need shell/tools at runtime
          ├── Minimise size + musl is OK?     → alpine:3.19                   (~7 MB)
          └── Debian-compatible packages?     → python:3.x-slim-bookworm     (~130 MB)
                                              → node:20-slim                 (~70 MB)
                                              → debian:12-slim               (~30 MB)
```

---

## Distroless Images

Maintained by Google. No shell, no package manager, minimal OS libraries.

| Image | Tag | Notes |
|---|---|---|
| Static (no libc) | `gcr.io/distroless/static-debian12` | Go/Rust static only |
| Base (glibc) | `gcr.io/distroless/base-debian12` | C/C++, dynamic Go |
| Node.js 20 | `gcr.io/distroless/nodejs20-debian12` | Node runtime |
| Java 21 | `gcr.io/distroless/java21-debian12` | JVM apps |
| Python 3.11 | `gcr.io/distroless/python3-debian12` | CPython (no pip at runtime) |

**Debug variant:** append `-debug` for a busybox shell during troubleshooting only:
```
gcr.io/distroless/base-debian12:debug
```
Never ship `:debug` to production.

**Nonroot variant:** append `-nonroot` to enforce UID 65532 even if USER is not set:
```
gcr.io/distroless/static-debian12:nonroot
```

---

## Alpine

- Based on musl libc — most C extensions need recompilation.
- Great for: shell scripts, simple Go binaries compiled with musl, minimal tools.
- Avoid for: Python packages with C extensions (numpy, pandas), Java.
- Always pin: `alpine:3.19`, not `alpine:latest`.

```dockerfile
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
```

- `ca-certificates` — needed for HTTPS calls from the container.
- `tzdata` — needed if the app reads the local timezone.

---

## Slim Debian / Ubuntu

Use when you need `apt` packages or glibc-dependent binaries but want a smaller image than full Debian.

| Image | Notes |
|---|---|
| `debian:12-slim` | ~30 MB, no extra packages |
| `ubuntu:24.04` | ~30 MB, more package availability |
| `node:20-slim` | Node + npm, Debian slim base |
| `python:3.12-slim-bookworm` | Python + pip, Debian slim |

Always clean up after `apt-get`:
```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
 && rm -rf /var/lib/apt/lists/*
```

---

## Tag Pinning Strategy

### ❌ Never use
```dockerfile
FROM node:latest          # mutable, breaks silently
FROM python:3             # floats to latest 3.x
FROM ubuntu               # same as :latest
```

### ✅ Use minor-version tags (recommended)
```dockerfile
FROM node:20-bookworm     # stable channel, security patches applied
FROM python:3.12-slim-bookworm
FROM golang:1.22-bookworm
```

### ✅ Use digest pins for maximum reproducibility (CI/CD)
```dockerfile
FROM node:20-bookworm@sha256:abc123...
```

Generate the digest:
```bash
docker pull node:20-bookworm
docker inspect node:20-bookworm --format '{{index .RepoDigests 0}}'
```

---

## CVE Scanning

Scan images before pushing to a registry:

```bash
# Trivy (recommended)
trivy image myapp:latest

# Docker Scout (built into Docker Desktop)
docker scout cves myapp:latest

# Grype
grype myapp:latest
```

Integrate into CI:
```yaml
# GitHub Actions
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:latest
    exit-code: '1'
    severity: CRITICAL,HIGH
```

---

## Image Size Comparison (typical Node.js app)

| Base | Final Size | Shell | CVEs (approx) |
|---|---|---|---|
| `node:20` | ~1.1 GB | yes | 300+ |
| `node:20-slim` | ~230 MB | yes | ~100 |
| `node:20-alpine` | ~130 MB | sh | ~15 |
| `distroless/nodejs20` | ~100 MB | no | <10 |

Distroless wins on security surface. Alpine wins on pure size when a shell is acceptable.
