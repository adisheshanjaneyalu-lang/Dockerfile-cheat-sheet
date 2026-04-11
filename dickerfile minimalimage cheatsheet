# Dockerfile Minimal Base Image Cheat Sheet
> Which runtime image to use for every language — production-grade guide

---

## The Golden Rule

```
Build Stage    →  big image  (has compiler, tools, git)
Runtime Stage  →  tiny image (just enough to RUN the binary)
```

The runtime image is what actually ships to production (ECR, DockerHub).
Smaller = faster pull, less attack surface, cheaper storage.

---

## Quick Decision Tree

```
What language is it?
│
├── Go / Rust          →  alpine:3.19  OR  scratch  OR  distroless
│
├── React / Vue / Angular (static files)
│                      →  nginx:alpine
│
├── Java               →  eclipse-temurin:VERSION-jre-alpine
│
├── Python             →  python:VERSION-alpine  (needs interpreter)
│
├── Node.js / Express  →  node:VERSION-alpine    (needs interpreter)
│
└── Ruby               →  ruby:VERSION-alpine    (needs interpreter)
```

---

## The 4 Minimal Runtime Images (Explained Simply)

### 1. `alpine:3.19`  — THE MOST COMMON (use this when unsure)

```
Size      : ~5 MB
Has       : basic Linux OS, shell (sh), ls, cat, wget
Cannot do : nothing — it's a full tiny OS
```

**Use when:**
- Your Go binary needs CA certificates for HTTPS calls
- Your app needs timezone data (`tzdata`)
- You need to `docker exec` into container for debugging
- Python/Node apps (they need their interpreter anyway)

```dockerfile
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app/mybinary .
CMD ["./mybinary"]
```

**Real example — ShopVerse Go backend:**
```dockerfile
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
RUN adduser -D -s /bin/sh appuser
WORKDIR /app
COPY --from=builder /app/shopverse-backend .
USER appuser
EXPOSE 8080
CMD ["./shopverse-backend"]
```

---

### 2. `scratch`  — ABSOLUTE MINIMUM (0 MB — empty box)

```
Size      : 0 MB — literally nothing
Has       : nothing at all
Cannot do : no shell, no ls, no ping, no debugging
```

**Use when:**
- Pure Go binary compiled with `CGO_ENABLED=0`
- Maximum security — no shell = no way to execute anything if hacked
- You never need to debug inside the container

```dockerfile
FROM scratch
# Must manually copy SSL certs (no apk to install them!)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/mybinary .
CMD ["/mybinary"]
```

**Important:** If your Go app makes HTTPS calls, you MUST copy CA certs manually.
If you use `scratch` and forget certs → all HTTPS requests fail silently.

**Real example — Ultra secure Go service:**
```dockerfile
# Stage 1
FROM golang:1.24-alpine AS builder
RUN apk add --no-cache git ca-certificates tzdata
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o app ./cmd/main.go

# Stage 2
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /app/app .
CMD ["/app"]
```

---

### 3. `gcr.io/distroless/static-debian12`  — MOST SECURE WITH SOME FEATURES

```
Size      : ~2 MB
Has       : bare minimum to run a binary (libc, SSL certs built-in)
Cannot do : no shell, no package manager, no OS tools
```

**Use when:**
- Security is the top priority (production microservices)
- Go binary that needs libc but no full OS
- Your company has strict security policies
- Used by Google internally for all their services

```dockerfile
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/mybinary .
CMD ["/mybinary"]
```

**Distroless variants:**
| Image | Use for |
|-------|---------|
| `gcr.io/distroless/static-debian12` | Go binaries (CGO_ENABLED=0) |
| `gcr.io/distroless/base-debian12` | Go binaries that need glibc |
| `gcr.io/distroless/java17-debian12` | Java 17 apps |
| `gcr.io/distroless/python3-debian12` | Python 3 apps |
| `gcr.io/distroless/nodejs20-debian12` | Node.js 20 apps |

**Real example — Your Shivam Hospital pattern:**
```dockerfile
FROM gcr.io/distroless/static-debian12
WORKDIR /app
COPY --from=builder /app/hospital-backend .
EXPOSE 8000
CMD ["/app/hospital-backend"]
```

---

### 4. `nginx:alpine`  — FOR ALL FRONTEND APPS

```
Size      : ~23 MB
Has       : Alpine Linux + Nginx web server
Cannot do : run backend code, talk to databases
```

**Use when:**
- React (Vite → dist/ folder)
- React (CRA → build/ folder)
- Vue, Angular, Svelte
- Any app that compiles to static HTML/CSS/JS files

```dockerfile
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/dist /usr/share/nginx/html    # Vite
# COPY --from=builder /app/build /usr/share/nginx/html  # CRA
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Vite vs CRA — build output folder:**
| Tool | Build command | Output folder |
|------|--------------|---------------|
| Vite | `npm run build` | `dist/` |
| Create React App (CRA) | `npm run build` | `build/` |
| Next.js | `npm run build` | `.next/` |
| Angular | `ng build` | `dist/project-name/` |

---

## Full Language Reference Table

| Language | Build Image | Runtime Image | Why this runtime? | Image size |
|----------|------------|---------------|-------------------|------------|
| **Go** | `golang:1.24-alpine` | `alpine:3.19` | Needs OS for certs/tz | ~12 MB |
| **Go** | `golang:1.24-alpine` | `scratch` | Zero overhead, no shell | ~8 MB |
| **Go** | `golang:1.24-alpine` | `gcr.io/distroless/static-debian12` | Secure, certs included | ~10 MB |
| **React (Vite)** | `node:18-alpine` | `nginx:alpine` | Serves static dist/ files | ~25 MB |
| **React (CRA)** | `node:18-alpine` | `nginx:alpine` | Serves static build/ files | ~25 MB |
| **Vue / Angular** | `node:18-alpine` | `nginx:alpine` | Serves static files | ~25 MB |
| **Python** | `python:3.11-alpine` | `python:3.11-alpine` | Needs Python interpreter | ~55 MB |
| **FastAPI/Flask** | `python:3.11-alpine` | `python:3.11-alpine` | Needs Python to run | ~55 MB |
| **Node.js/Express** | `node:18-alpine` | `node:18-alpine` | Needs Node to run | ~115 MB |
| **Java Spring Boot** | `maven:3.9-alpine` | `eclipse-temurin:17-jre-alpine` | JRE runs .jar, no compiler | ~85 MB |
| **Java Gradle** | `gradle:8-alpine` | `eclipse-temurin:17-jre-alpine` | JRE runs .jar, no compiler | ~85 MB |
| **Rust** | `rust:1.75-alpine` | `alpine:3.19` OR `scratch` | Binary is self-contained | ~8 MB |
| **Ruby on Rails** | `ruby:3.2-alpine` | `ruby:3.2-alpine` | Needs Ruby interpreter | ~65 MB |

---

## Compiled vs Interpreted — The Core Concept

This is WHY different languages use different runtime images:

```
COMPILED languages → binary runs WITHOUT original language installed
────────────────────────────────────────────────────────────────────
  Go, Rust, C++

  Code ──[compiler]──► binary ──► runs on bare OS
                         │
                         └── only needs: OS + libraries
                         └── use: alpine / scratch / distroless ✅
                         └── HUGE size savings possible!

INTERPRETED languages → NEED the language runtime to execute
────────────────────────────────────────────────────────────────────
  Python, Node.js, Ruby

  Code ──► interpreter reads it line by line ──► executes
             │
             └── Python / Node / Ruby must be installed
             └── cannot use scratch or plain alpine
             └── use same language image (just alpine variant)
             └── size savings are smaller
```

---

## Size Comparison — Why This Matters for K8s/EKS

```
Image                               Size      Notes
──────────────────────────────────────────────────────────────
golang:1.24-alpine (build image)    ~300 MB   Has full Go compiler
node:18 (no alpine)                 ~900 MB   Full Ubuntu + Node
node:18-alpine                      ~165 MB   Alpine + Node
python:3.11 (no alpine)             ~900 MB   Full Debian + Python
python:3.11-alpine                  ~ 55 MB   Alpine + Python
nginx:alpine (frontend runtime)     ~ 23 MB   Alpine + Nginx
eclipse-temurin:17-jre-alpine       ~ 85 MB   Alpine + Java JRE only
alpine:3.19 (Go runtime)            ~  5 MB   Just OS
gcr.io/distroless/static            ~  2 MB   Bare minimum
scratch                             ~  0 MB   Absolutely nothing
──────────────────────────────────────────────────────────────
Go multi-stage final image          ~ 10 MB   96% smaller than build!
```

On EKS with 10 pods pulling 300MB vs 10MB:
- 300MB × 10 = 3000 MB pulled from ECR
-  10MB × 10 =  100 MB pulled from ECR
→ Faster pod startup, lower ECR transfer costs, less memory pressure on nodes

---

## Complete Multi-Stage Examples

### Go Backend (alpine runtime)
```dockerfile
FROM golang:1.24-alpine AS builder
RUN apk add --no-cache git
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o app ./cmd/main.go

FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
RUN adduser -D -s /bin/sh appuser
WORKDIR /app
COPY --from=builder /app/app .
RUN chown appuser:appuser ./app
USER appuser
EXPOSE 8080
CMD ["./app"]
```

### React Vite Frontend (nginx runtime)
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Python FastAPI (python:alpine runtime)
```dockerfile
FROM python:3.11-alpine AS builder
RUN apk add --no-cache gcc musl-dev libpq-dev
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-alpine
RUN apk add --no-cache libpq
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Node.js Express API (node:alpine runtime)
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --only=production
COPY . .
RUN adduser -D appuser
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

### Java Spring Boot (JRE runtime)
```dockerfile
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN adduser -D appuser
COPY --from=builder /app/target/*.jar app.jar
USER appuser
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

---

## When to Use Which — Simple Decision

| Situation | Use |
|-----------|-----|
| Go app, want smallest possible | `scratch` |
| Go app, need to debug sometimes | `alpine:3.19` |
| Go app, high security production | `gcr.io/distroless/static-debian12` |
| React/Vue/Angular frontend | `nginx:alpine` |
| Python app | `python:VERSION-alpine` |
| Node.js backend | `node:VERSION-alpine` |
| Java app | `eclipse-temurin:VERSION-jre-alpine` |
| Not sure / first time | `alpine:3.19` — safe default for everything |

---

## Production Checklist

- [ ] Runtime image is different (smaller) from build image
- [ ] Using `-alpine` variant wherever possible
- [ ] `CGO_ENABLED=0` set for Go builds (required for scratch/distroless)
- [ ] Non-root user created (`adduser -D appuser` + `USER appuser`)
- [ ] Secrets NOT hardcoded — passed via env vars at runtime
- [ ] `.dockerignore` exists (excludes `node_modules`, `.env`, `.git`)
- [ ] CA certificates included if app makes HTTPS calls
- [ ] `EXPOSE` port documented
- [ ] `CMD` uses exec form: `CMD ["./app"]` not `CMD ./app`

---

## Quick Reference Card

```
Language     File to look for    Build image              Runtime image
──────────────────────────────────────────────────────────────────────────
Go           go.mod              golang:VERSION-alpine    alpine:3.19
React(Vite)  package.json        node:VERSION-alpine      nginx:alpine
React(CRA)   package.json        node:VERSION-alpine      nginx:alpine
Python       requirements.txt    python:VERSION-alpine    python:VERSION-alpine
Node.js      package.json        node:VERSION-alpine      node:VERSION-alpine
Java Maven   pom.xml             maven:VERSION-alpine     eclipse-temurin:VER-jre-alpine
Java Gradle  build.gradle        gradle:VERSION-alpine    eclipse-temurin:VER-jre-alpine
Rust         Cargo.toml          rust:VERSION-alpine      alpine:3.19
Ruby         Gemfile             ruby:VERSION-alpine      ruby:VERSION-alpine
──────────────────────────────────────────────────────────────────────────
Not sure?    any                 use language:VERSION-alpine for both stages
```

---

*Cheat sheet by Adishesh | ShopVerse / DevSecOps-Zero-to-Hero project*
