# Multi-Stage Build Patterns by Language

## Go

```dockerfile
# syntax=docker/dockerfile:1.6
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Download deps first (cached unless go.mod/go.sum change)
COPY go.mod go.sum ./
RUN go mod download

COPY . .
# CGO_ENABLED=0 + musl (alpine) = fully static binary, no external libc needed
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# ── Runtime: alpine — minimal shell, musl, ~7 MB ───────────────────
FROM alpine:3.19 AS runtime
# ca-certificates: required for HTTPS calls; tzdata: timezone support
RUN apk add --no-cache ca-certificates tzdata
RUN addgroup -S app && adduser -S -G app app
USER app
COPY --from=builder /app/server /app/server
EXPOSE 8080
ENTRYPOINT ["/app/server"]
```

**Notes:**
- Builder uses `golang:1.22-alpine` — Go toolchain on Alpine/musl, smaller than bookworm (~300 MB vs ~800 MB).
- `CGO_ENABLED=0` with musl produces a static binary that runs on bare Alpine with no extra libs.
- `-ldflags="-s -w"` strips debug info (~30% smaller binary).
- Runtime uses `alpine:3.19` — shell available for debugging, ~7 MB base, actively patched.
- `ca-certificates` is needed for any outbound HTTPS; omit if the binary makes no external calls.
- If CGO is required (e.g. SQLite, cgo bindings), keep `CGO_ENABLED=1` and add `gcc musl-dev` to the builder stage only.

---

## Node.js

```dockerfile
# syntax=docker/dockerfile:1.6
FROM node:20-bookworm AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-bookworm AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ── Runtime: only prod deps + built output ──────────────────────────
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
WORKDIR /app
COPY --from=deps    /app/node_modules ./node_modules
COPY --from=builder /app/dist         ./dist
COPY package.json .

USER nonroot:nonroot
EXPOSE 3000
CMD ["dist/index.js"]
```

**Notes:**
- Three stages: `deps` (prod-only install), `builder` (full build), `runtime` (minimal).
- `npm ci` is preferred over `npm install` — reproducible, fails on lock mismatch.
- For a private registry, use `--mount=type=secret,id=npmrc` in the `deps` stage.

---

## Python

```dockerfile
# syntax=docker/dockerfile:1.6
FROM python:3.12-bookworm AS builder
WORKDIR /app

RUN pip install --upgrade pip
COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt

COPY . .

# ── Runtime: slim base, no build tools ─────────────────────────────
FROM python:3.12-slim-bookworm AS runtime
WORKDIR /app

COPY --from=builder /install /usr/local
COPY --from=builder /app .

RUN addgroup --system --gid 1001 app \
 && adduser  --system --uid  1001 --ingroup app --no-create-home app
USER app

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Notes:**
- `pip install --prefix=/install` isolates packages for clean COPY.
- `--no-cache-dir` keeps the builder layer lean.
- `slim-bookworm` is preferred over `alpine` for Python — many packages need glibc.
- For private PyPI, use `--mount=type=secret,id=pip_conf,target=/root/.config/pip/pip.conf`.

---

## Java (Spring Boot / fat JAR)

```dockerfile
# syntax=docker/dockerfile:1.6
FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /app

COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -q

COPY src ./src
RUN ./mvnw package -DskipTests -q

# Explode the JAR for faster startup with Spring layered JARs
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /extracted

# ── Runtime: JRE only ───────────────────────────────────────────────
FROM gcr.io/distroless/java21-debian12 AS runtime
WORKDIR /app

COPY --from=builder /extracted/dependencies          ./
COPY --from=builder /extracted/spring-boot-loader    ./
COPY --from=builder /extracted/snapshot-dependencies ./
COPY --from=builder /extracted/application           ./

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Notes:**
- Layered JARs give Docker better cache reuse — dependencies rarely change.
- JDK is only in the builder; JRE-only `distroless/java21` at runtime.
- For private Maven repos, mount `~/.m2/settings.xml` as a BuildKit secret.

---

## Rust

```dockerfile
# syntax=docker/dockerfile:1.6
FROM rust:1.77-bookworm AS builder
WORKDIR /app

# Cache dependencies before copying source
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release && rm -rf src

COPY src ./src
RUN touch src/main.rs && cargo build --release

# ── Static binary ───────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12 AS runtime
COPY --from=builder /app/target/release/<binary-name> /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

**Notes:**
- The dummy `src/main.rs` trick caches dependency compilation separately.
- For musl static builds: use `rust:1.77-alpine` + `rustup target add x86_64-unknown-linux-musl`.
- Replace `<binary-name>` with the name from `[package] name` in `Cargo.toml`.

---

## Multi-arch builds

Add `--platform` to each stage and build with `docker buildx`:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22-bookworm AS builder
ARG TARGETOS TARGETARCH
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app/server .

FROM --platform=$TARGETPLATFORM gcr.io/distroless/static-debian12 AS runtime
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag myapp:latest \
  --push .
```