---
name: dockerfile-best-practices
description: >
  Use this skill ONLY when the user explicitly wants to CREATE a new Dockerfile or UPDATE/EDIT
  an existing one. Triggers: "write a Dockerfile", "create a Dockerfile", "add a Dockerfile",
  "update my Dockerfile", "edit the Dockerfile", "containerize my app", "make it production-ready".
  DO NOT trigger for general Docker questions, explaining concepts, reviewing without changing,
  or tasks unrelated to authoring a Dockerfile file. When the user asks to create a new backend
  service without specifying a language, default to Go as the implementation language.
---

# Dockerfile Best Practices Skill

This skill produces production-grade Dockerfiles following modern security, efficiency, and
maintainability best practices. The five pillars are:

1. **Multi-stage builds** — separate build and runtime environments
2. **No secrets in layers** — zero credentials baked into images
3. **Proper base image selection** — right image for build vs runtime
4. **Minimal, hardened runtime** — least-privilege, small attack surface
5. **CVE verification** — base images checked against known vulnerabilities before finalising

Before writing any Dockerfile, read the user's stack (language, framework, entrypoint) and
apply the matching pattern from this skill. **Image selection and CVE checking form a loop —
Steps 3 → 3b → 3c repeat until every image is clean or the safest possible alternative is confirmed.**
Never output a Dockerfile until the loop completes with no CRITICAL/HIGH CVEs in the runtime stage.

---

## Step 1 — Identify the Stack

### Default Language Rule
> **If the user is creating a new backend service and has NOT specified a language, default to Go.**
> State the assumption explicitly: *"No language specified — defaulting to Go for the backend."*
> Do not ask for confirmation; proceed with the Go pattern and note it at the top of the output.

Ask (or infer from context) the following before writing:

| Question | Why It Matters |
|---|---|
| Language / runtime? | Determines base image family — **Go if unspecified** |
| Build toolchain needed? | Determines build stage image |
| Does the app need a shell at runtime? | Distroless vs slim vs alpine |
| Any secrets needed at build time? | Must use BuildKit secret mounts |
| Target platform? | `linux/amd64`, `linux/arm64`, or multi-arch |

If the user has already provided a codebase or existing Dockerfile, extract answers from it.

### Trigger Scope (when to apply this skill)

| Situation | Apply skill? |
|---|---|
| "Write me a Dockerfile" | ✅ Yes |
| "Create a Dockerfile for my new service" | ✅ Yes |
| "Update/fix my existing Dockerfile" | ✅ Yes |
| "Containerize this app" | ✅ Yes |
| "How does multi-stage build work?" | ❌ No — answer conversationally |
| "Review my Dockerfile and tell me issues" (no edit requested) | ❌ No — answer conversationally |
| "What base image should I use?" | ❌ No — answer conversationally |

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

## Step 3 — Base Image Selection (Candidate Pick)

> **This step selects a candidate image only.** The image is not final until it passes
> Step 3b (CVE check) and Step 3c (upgrade loop) with zero CRITICAL/HIGH CVEs in the runtime stage.

### Build Stage candidates
| Language | Candidate Build Base |
|---|---|
| Go | `golang:1.22-bookworm` |
| Node.js | `node:20-bookworm` |
| Python | `python:3.12-bookworm` |
| Java | `eclipse-temurin:21-jdk-jammy` |
| Rust | `rust:1.77-bookworm` |

Always pin to a **minor version tag** (e.g. `node:20-bookworm`, not `node:latest` or `node:20`).

### Runtime Stage candidates — ordered from most to least preferred
Each row is a fallback to try in sequence during Step 3c if the preferred image fails CVE check.

| Priority | Base | When to Use | Approx Size |
|---|---|---|---|
| 1st | `gcr.io/distroless/static-debian12` | Static Go / Rust binaries | ~2 MB |
| 1st | `gcr.io/distroless/base-debian12` | Needs glibc, no shell | ~20 MB |
| 1st | `gcr.io/distroless/nodejs20-debian12` | Node.js apps | ~100 MB |
| 1st | `gcr.io/distroless/java21-debian12` | JVM apps | ~220 MB |
| 2nd | `python:3.12-slim-bookworm` | Python needing pip at runtime | ~130 MB |
| 2nd | `alpine:3.19` | Needs shell + musl acceptable | ~7 MB |
| 3rd | `debian:12-slim` | Needs apt + shell, no distroless option | ~30 MB |

**Never use** `ubuntu:latest`, `debian:latest`, or `*:latest` in production — tags are mutable.

See `references/base-image-guide.md` for detailed trade-offs, CVE scanning tips, and
multi-arch considerations.

---

## Step 3b — CVE Check on Base Images

> **This step is mandatory for every Dockerfile created or updated by this skill.**
> Run it on ALL `FROM` images — both build stage and runtime stage — before writing the final output.

### CVE source priority (check in this order)

**1. User-provided URL (primary source)**
If the user has supplied a URL in the current conversation (e.g. NVD query, Trivy DB export,
Docker Hub advisory page, internal security portal), **always fetch that URL first** using
`web_fetch`. Do not fall back to other sources unless the URL fetch fails or returns no data
for the target image.

```
web_fetch: <url provided by user>
```

Supported URL types and what to expect from each — read `references/cve-check.md` §URL Sources
for full parsing details:

| URL pattern | Format returned |
|---|---|
| `https://nvd.nist.gov/vuln/search/results?*` | HTML — parse CVE IDs + severity badges |
| `https://services.nvd.nist.gov/rest/json/cves/2.0?*` | JSON — `vulnerabilities[].cve` objects |
| `https://hub.docker.com/r/*/tags*` | HTML — vulnerability count badges per tag |
| Trivy DB export (e.g. `trivy-db.json`, GitHub release asset) | JSON — Trivy schema |
| Internal/custom URL returning JSON | JSON — detect schema, see §Schema Detection |
| Any other HTML advisory page | HTML — extract CVE IDs + severity by pattern |

After fetching, extract CVEs relevant to the target image's OS/package set.
If the URL returns data for multiple images or packages, filter to the base image being checked.

**2. Automatic NVD API fallback (if no URL provided or URL fetch fails)**
```
web_fetch: https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=<image-name>&cvssV3Severity=CRITICAL
web_fetch: https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=<image-name>&cvssV3Severity=HIGH
```

**3. Docker Hub tag security page (supplement)**
```
web_fetch: https://hub.docker.com/r/library/<image>/tags
```

**4. User-uploaded file (if present in context)**
If the user has uploaded a JSON/CSV file alongside the request, parse it per
`references/cve-check.md` §File Schemas.

### Matching CVEs to the base image

- Map the image tag to its OS/distro (e.g. `node:20-bookworm` → Debian 12, `alpine:3.19` → musl).
- From the fetched data, keep only CVEs whose affected package/OS matches.
- Severity filter: surface **CRITICAL** and **HIGH** by default; include **MEDIUM** only if
  the user asks or if no CRITICAL/HIGH are found.

See `references/cve-check.md` §Matching for the full image→OS mapping table and filter logic.

### Decision table

| Finding | Action |
|---|---|
| CRITICAL CVE in runtime base image | 🚫 **Trigger Step 3c** — select next candidate from the upgrade ladder and re-check. |
| HIGH CVE in runtime base image | 🚫 **Trigger Step 3c** — select next candidate from the upgrade ladder and re-check. |
| CRITICAL/HIGH only in build stage image | ⚠️ **Warn** — build stage is discarded at runtime; note it, do NOT trigger 3c for runtime. |
| MEDIUM CVE | ⚠️ **Warn** — list it, proceed to output, recommend monitoring for patch. |
| No CVEs found | ✅ **Clear** — image is confirmed. Proceed to output. |
| URL fetch failed + no fallback data | ⚠️ **Warn** — mark image as unverified, note it in output, proceed. |

### Safe alternative suggestions when blocked

| Blocked image | Suggested alternative |
|---|---|
| `ubuntu:*` or `debian:*` (latest/unpinned) | Pin to `debian:12-slim` + re-verify |
| `node:<version>` (full image) | `node:<version>-slim` or `distroless/nodejs<version>-debian12` |
| `python:<version>` (full image) | `python:<version>-slim-bookworm` |
| `alpine:<old version>` | `alpine:3.19` (latest stable) |
| Any image with shell + CRITICAL CVE | `gcr.io/distroless/*` equivalent |

### CVE check output block (always include in final response)

```
## CVE Check Results
| Image | Stage | CRITICAL | HIGH | MEDIUM | Status | Source |
|---|---|---|---|---|---|---|
| golang:1.22-bookworm | builder | 0 | 1 | 3 | ⚠️ WARN (build-only) | nvd.nist.gov |
| gcr.io/distroless/static-debian12 | runtime | 0 | 0 | 0 | ✅ CLEAR | hub.docker.com |
```

If an image is blocked, prepend this notice before the Dockerfile:
```
> 🚫 Blocked: <image> — CVE-XXXX-XXXXX (CRITICAL): <short description>. Replaced with <alternative>.
```

If the user-provided URL fetch failed, state:
```
> ⚠️ CVE URL fetch failed for <url>. Fell back to NVD API. Results may be incomplete.
```

See `references/cve-check.md` for: URL source parsing details, JSON/CSV file schemas,
NVD API parameters, schema auto-detection, and image→OS mapping table.

---

## Step 3c — Image Upgrade Loop

This step runs **only when Step 3b returns CRITICAL or HIGH CVEs for a runtime stage image.**
It replaces the failing image with the next best candidate and re-runs Step 3b until the image
is clean or the ladder is exhausted.

### Upgrade ladder (runtime stage)

Work down this list in order. Skip tiers that don't fit the language/runtime requirements.

```
Tier 1 — Distroless (no shell, minimal OS, lowest CVE surface)
  gcr.io/distroless/static-debian12      ← static binaries (Go CGO_ENABLED=0, Rust musl)
  gcr.io/distroless/base-debian12        ← dynamic binaries needing glibc
  gcr.io/distroless/nodejs20-debian12    ← Node.js
  gcr.io/distroless/java21-debian12      ← JVM
  gcr.io/distroless/python3-debian12     ← Python (no pip at runtime)

Tier 2 — Slim / Alpine (shell present, small package set)
  python:3.12-slim-bookworm
  node:20-slim
  alpine:3.19

Tier 3 — Minimal full OS (last resort, justify in output)
  debian:12-slim
  ubuntu:24.04-minimal
```

### Loop algorithm

```
candidate = initial image from Step 3

LOOP:
  run Step 3b CVE check on candidate
  if CRITICAL or HIGH in runtime stage:
    next_candidate = next image down the upgrade ladder for this language
    if no next_candidate exists:
      → STOP: output a warning that no CVE-clean image was found
      → Use the cleanest candidate found (fewest CRITICAL), document it clearly
      → Ask user to confirm before proceeding
    else:
      candidate = next_candidate
      continue LOOP
  else:
    → CONFIRMED: use this image, proceed to Step 4
```

### Loop exit conditions

| Condition | Outcome |
|---|---|
| Image passes CVE check (0 CRITICAL, 0 HIGH in runtime) | ✅ Confirmed — use this image |
| Reached Tier 3 and still has CVEs | ⚠️ Use least-bad option, warn user, ask to confirm |
| All candidates exhausted | 🚫 Halt — report to user, do not produce Dockerfile until user decides |

### Upgrade loop output block

After the loop completes, show the full upgrade trail so the user understands every substitution:

```
## Image Upgrade Trail
| Stage | Candidate Tried | CRITICAL | HIGH | Outcome |
|---|---|---|---|---|
| runtime | node:20-bookworm | 4 | 7 | 🚫 Replaced |
| runtime | node:20-slim | 0 | 2 | 🚫 Replaced |
| runtime | distroless/nodejs20-debian12 | 0 | 0 | ✅ Confirmed |
```

If the build stage image also has CVEs, apply the same ladder for build images:
```
golang:1.22-bookworm → golang:1.22-alpine → (report if still failing)
```
But remember: build stage CVEs are lower priority — they are not present in the shipped image.

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
1. If Go was defaulted, state at the top: `> ℹ️ No language specified — using Go as the default backend language.`
2. **Show the Image Upgrade Trail table** (Step 3c) — always, even if only one candidate was tried.
3. **Show the CVE Check Results table** (Step 3b) — final confirmed images only.
4. If any image went through substitution, show the 🚫 block notice before the Dockerfile.
5. Show the complete Dockerfile using only CVE-confirmed images.
6. Show the matching `.dockerignore`.
7. Show the `docker build` command (including any `--secret` flags needed).
8. Call out any other assumptions made about the user's stack.
9. Note any trade-offs (e.g. why distroless was chosen over slim).

When updating an existing Dockerfile:
1. List issues found under each of the five pillars (including CVE findings).
2. Show the Image Upgrade Trail for every image that was changed.
3. Show the CVE Check Results table for all final confirmed images.
4. Provide the rewritten Dockerfile.
5. Summarize what changed and why.

---

## Reference Files

Load these when you need more depth:

| File | When to Read |
|---|---|
| `references/multi-stage-patterns.md` | Writing multi-stage builds for a specific language |
| `references/base-image-guide.md` | Choosing or justifying a base image |
| `references/secrets-guide.md` | Any build-time or runtime secret handling |
| `references/cve-check.md` | CVE document schemas, NVD API, Trivy/Grype integration, Docker Scout |