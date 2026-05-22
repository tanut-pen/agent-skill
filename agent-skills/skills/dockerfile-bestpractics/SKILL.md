---
name: dockerfile-best-practices
description: >
  Use this skill whenever the user asks to create, write, review, fix, or optimize a Dockerfile,
  container image, or Docker-related configuration. Triggers include: "write a Dockerfile",
  "containerize my app", "Docker multi-stage", "Docker best practices", "reduce image size",
  "Docker secrets", "secure Dockerfile", "Docker base image", "production Dockerfile",
  or any request involving building container images. Also trigger when the user uploads
  or pastes a Dockerfile and asks for improvements, security review, or optimization.
  Always use this skill for any Docker image authoring task — even simple ones, as
  best practices around secrets, base images, and build stages are easy to get wrong.
---

# Dockerfile Best Practices Skill

This skill produces production-grade Dockerfiles following modern security, efficiency, and
maintainability best practices. The four pillars are:

1. **Multi-stage builds** — separate build and runtime environments
2. **No secrets in layers** — zero credentials baked into images
3. **Proper base image selection** — right image for build vs runtime
4. **Minimal, hardened runtime** — least-privilege, small attack surface

Before writing any Dockerfile, read the user's stack (language, framework, entrypoint) and
apply the matching pattern from this skill. When reviewing an existing Dockerfile, check
all four pillars and report findings before rewriting.

---

## Step 1 — Identify the Stack

Ask (or infer from context) the following before writing:

| Question | Why It Matters |
|---|---|
| Language / runtime? | Determines base image family |
| Build toolchain needed? | Determines build stage image |
| Does the app need a shell at runtime? | Distroless vs slim vs alpine |
| Any secrets needed at build time? | Must use BuildKit secret mounts |
| Target platform? | `linux/amd64`, `linux/arm64`, or multi-arch |

If the user has already provided a codebase or existing Dockerfile, extract answers from it.

---

## Step 2 — Multi-Stage Build Pattern

Always use at least two stages: **builder** and **runtime**.

```dockerfile
# ── Stage 1: builder ───────────────────────────────────────────────
FROM <build-base> AS builder
WORKDIR /app

# Install ONLY build dependencies here
COPY <dependency-manifest> .
RUN <install-deps>

COPY . .
RUN <compile-or-build>

# ── Stage 2: runtime ───────────────────────────────────────────────
FROM <runtime-base> AS runtime
WORKDIR /app

# Copy ONLY the built artifact(s) — nothing from the build toolchain
COPY --from=builder /app/<output> .

# Non-root user
RUN addgroup --system app && adduser --system --ingroup app app
USER app

EXPOSE <port>
ENTRYPOINT ["<binary>"]
```

**Rules:**
- Never use `FROM <build-base>` as the final stage for production images.
- `COPY --from=builder` copies only what the runtime actually needs.
- No compilers, test frameworks, or dev dependencies in the final image.
- Label stages with `AS <name>` for clarity and selective builds.

See `references/multi-stage-patterns.md` for per-language examples (Go, Node, Python, Java, Rust).

---

## Step 3 — Base Image Selection

### Build Stage
| Language | Recommended Build Base |
|---|---|
| Go | `golang:1.22-bookworm` |
| Node.js | `node:20-bookworm` |
| Python | `python:3.12-bookworm` |
| Java | `eclipse-temurin:21-jdk-jammy` |
| Rust | `rust:1.77-bookworm` |

Always pin to a **minor version tag** (e.g. `node:20-bookworm`, not `node:latest` or `node:20`).

### Runtime Stage (choose one per use case)
| Base | When to Use | Approx Size |
|---|---|---|
| `gcr.io/distroless/static-debian12` | Static Go / Rust binaries | ~2 MB |
| `gcr.io/distroless/base-debian12` | Needs glibc, no shell | ~20 MB |
| `gcr.io/distroless/nodejs20-debian12` | Node.js apps | ~100 MB |
| `gcr.io/distroless/java21-debian12` | JVM apps | ~220 MB |
| `python:3.12-slim-bookworm` | Python apps needing pip at runtime | ~130 MB |
| `alpine:3.19` | Needs a shell + musl is acceptable | ~7 MB |

**Never use** `ubuntu:latest`, `debian:latest`, or `*:latest` in production — tags are mutable.

See `references/base-image-guide.md` for detailed trade-offs, CVE scanning tips, and
multi-arch considerations.

---

## Step 4 — No Secrets in Layers

### Absolute Rules
- **Never** use `ARG` or `ENV` for passwords, tokens, or API keys — they appear in `docker history`.
- **Never** `COPY` `.env`, `*.pem`, `*_key`, or credential files into the image.
- **Never** `RUN curl ... -H "Authorization: Bearer $TOKEN"` — the token ends up in the layer.

### Build-time secrets (e.g. private package registries)

Use **BuildKit secret mounts** — secrets are never written to any layer:

```dockerfile
# syntax=docker/dockerfile:1.6
FROM node:20-bookworm AS builder
WORKDIR /app
COPY package*.json .

# Secret is mounted read-only at /run/secrets/npmrc — not stored in image
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --omit=dev
```

Build with:
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

### Runtime secrets
Inject via environment variables **at container start** (not image build):
```bash
docker run -e DATABASE_URL="$DATABASE_URL" myapp:latest
# or use Docker Secrets (Swarm) / Kubernetes Secrets / cloud secret manager sidecars
```

See `references/secrets-guide.md` for SSH agent forwarding, private Git repos,
cloud KMS patterns, and `.dockerignore` essentials.

---

## Step 5 — Hardening Checklist

Apply all of the following to the **runtime stage**:

```dockerfile
# 1. Non-root user (create if distroless doesn't provide one)
RUN addgroup --system --gid 1001 app \
 && adduser  --system --uid  1001 --ingroup app --no-create-home app
USER app

# 2. Read-only filesystem where possible (set at runtime, not Dockerfile)
#    docker run --read-only --tmpfs /tmp myapp

# 3. No new privileges (set at runtime)
#    docker run --security-opt=no-new-privileges myapp

# 4. Explicit EXPOSE — document the port, don't bind to 0.0.0.0 unnecessarily
EXPOSE 8080

# 5. Use COPY not ADD (ADD can auto-extract archives and fetch URLs — avoid)
COPY --from=builder /app/server /app/server

# 6. Prefer ENTRYPOINT + CMD over CMD alone
ENTRYPOINT ["/app/server"]
CMD ["--port", "8080"]

# 7. Metadata labels
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.revision="<git-sha>"
```

---

## Step 6 — Layer Caching Optimization

Order instructions from **least to most frequently changed**:

```
1. FROM base image
2. System package installs (apt, apk)
3. Dependency manifest copy (package.json, go.mod, requirements.txt)
4. Dependency install (npm ci, go mod download, pip install)
5. Application source copy
6. Build command
```

Combine related `RUN` commands with `&&` to reduce layer count:
```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

---

## Step 7 — .dockerignore

Always create `.dockerignore` alongside the Dockerfile:

```
.git
.gitignore
.env*
*.md
node_modules/
__pycache__/
*.pyc
.pytest_cache/
dist/
build/
coverage/
.DS_Store
Dockerfile*
docker-compose*
```

---

## Output Format

When producing a Dockerfile:
1. Show the complete Dockerfile with inline comments explaining each decision.
2. Show the matching `.dockerignore`.
3. Show the `docker build` command (including any `--secret` flags needed).
4. Call out any assumptions made about the user's stack.
5. Note any trade-offs (e.g. alpine vs distroless).

When reviewing an existing Dockerfile:
1. List issues found under each of the four pillars.
2. Provide the rewritten Dockerfile.
3. Summarize what changed and why.

---

## Reference Files

Load these when you need more depth:

| File | When to Read |
|---|---|
| `references/multi-stage-patterns.md` | Writing multi-stage builds for a specific language |
| `references/base-image-guide.md` | Choosing or justifying a base image |
| `references/secrets-guide.md` | Any build-time or runtime secret handling |
