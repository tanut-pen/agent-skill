# Secrets Guide for Docker Builds

## The Golden Rule

> A secret that appears in ANY `RUN`, `ENV`, or `ARG` instruction is stored in the image layer — even if you `unset` it or delete the file afterward. Every layer is permanent.

---

## Method 1: BuildKit Secret Mounts (preferred for build-time secrets)

Available since Docker 18.09 with `DOCKER_BUILDKIT=1` (default in Docker 23+).

Secrets are mounted as read-only tmpfs files during a single `RUN` step. They are **never written to any layer**.

### Enable BuildKit syntax

Add this as the first line of your Dockerfile:
```dockerfile
# syntax=docker/dockerfile:1.6
```

### Private npm registry

```dockerfile
# syntax=docker/dockerfile:1.6
FROM node:20-bookworm AS builder
WORKDIR /app
COPY package*.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

### Private PyPI / pip

```dockerfile
RUN --mount=type=secret,id=pip_conf,target=/root/.config/pip/pip.conf \
    pip install --no-cache-dir -r requirements.txt
```

```bash
docker build --secret id=pip_conf,src=$HOME/.config/pip/pip.conf .
```

### Private Maven repository

```dockerfile
RUN --mount=type=secret,id=maven_settings,target=/root/.m2/settings.xml \
    ./mvnw package -DskipTests
```

```bash
docker build --secret id=maven_settings,src=$HOME/.m2/settings.xml .
```

### Inline secret from environment variable

```bash
echo "$MY_SECRET" | docker build --secret id=mysecret,src=- .
```

In Dockerfile:
```dockerfile
RUN --mount=type=secret,id=mysecret \
    SECRET=$(cat /run/secrets/mysecret) && \
    curl -H "Authorization: Bearer $SECRET" https://internal.api/resource
```

---

## Method 2: SSH Agent Forwarding (private Git repos)

Never clone private repos by embedding SSH keys in the image. Use SSH agent forwarding.

```dockerfile
# syntax=docker/dockerfile:1.6
FROM golang:1.22-bookworm AS builder

RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git /app/private-repo
```

```bash
# Ensure SSH agent has your key loaded
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519

docker build --ssh default .
```

For Docker Compose:
```yaml
services:
  builder:
    build:
      context: .
      ssh:
        - default
```

---

## Method 3: Runtime Secrets (not baked in at all)

For secrets the running container needs (DB passwords, API keys, etc.):

### Environment variables (simple deployments)
```bash
docker run \
  -e DATABASE_URL="postgres://user:pass@host/db" \
  -e API_KEY="$MY_API_KEY" \
  myapp:latest
```

Never hardcode these in the Dockerfile:
```dockerfile
# ❌ NEVER do this
ENV DATABASE_PASSWORD=supersecret
```

### Docker Swarm Secrets
```bash
echo "supersecret" | docker secret create db_password -
```

```yaml
# docker-compose.yml (Swarm mode)
services:
  app:
    image: myapp:latest
    secrets:
      - db_password
secrets:
  db_password:
    external: true
```

The secret is available at `/run/secrets/db_password` inside the container.

### Kubernetes Secrets
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: postgres://user:pass@host/db
```

```yaml
# Pod spec
envFrom:
  - secretRef:
      name: app-secrets
```

### Cloud Secret Managers
- **AWS**: Secrets Manager / Parameter Store — use IAM roles, not credentials in the image.
- **GCP**: Secret Manager — use Workload Identity.
- **Azure**: Key Vault — use Managed Identity.

---

## .dockerignore — Stop Secrets From Entering the Build Context

Even if you don't `COPY` them, files in the build context are sent to the Docker daemon. Always exclude sensitive files:

```
# Credentials and secrets
.env
.env.*
*.pem
*.key
*.p12
*.pfx
id_rsa
id_ed25519
*_rsa
*_ed25519
.ssh/
.aws/
.gcp/
credentials.json
service-account.json

# Dev tooling (not needed at runtime)
.git/
.gitignore
.github/
node_modules/
__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
dist/
build/
coverage/
.coverage
*.log

# Docker files themselves
Dockerfile*
docker-compose*
.dockerignore
```

---

## Auditing an Existing Image for Leaked Secrets

```bash
# Inspect image history for ENV/ARG values
docker history --no-trunc myapp:latest

# Scan for embedded secrets
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --scanners secret myapp:latest

# Alternatively, Trufflehog
docker run --rm trufflesecurity/trufflehog:latest docker --image myapp:latest
```

---

## Anti-Patterns to Reject

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| `ARG API_KEY` + `RUN curl -H "Authorization: $API_KEY"` | ARG values baked into layer | BuildKit `--mount=type=secret` |
| `ENV DB_PASS=hardcoded` | Visible in `docker history` | Runtime env var injection |
| `COPY .env .` | `.env` contents in image | Add `.env` to `.dockerignore` |
| `RUN echo $TOKEN > /tmp/token && <use it> && rm /tmp/token` | Layer still contains the file | BuildKit secret mount |
| SSH key in `COPY id_rsa /root/.ssh/` | Key in image, even if deleted later | SSH agent forwarding |
