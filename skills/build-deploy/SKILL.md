---
name: build-deploy
description: >-
  Docker build + push to any container registry, then deploy to Kubernetes via
  kubectl. Use when the user asks to build images, push to a registry, deploy
  to K8s, release a new version, bump image tags, write a Dockerfile, apply
  secrets from .env, or roll out a manifest. Covers the full pipeline:
  Dockerfile authoring → Docker build → registry push → K8s secret upsert →
  manifest apply.
---

# Docker Build, Push & Kubernetes Deploy

General-purpose skill for containerising an app and deploying it to Kubernetes.
Works for any stack, registry, or cluster.

---

## Step 0 — Write Dockerfiles (if needed)

### Detect what's needed
Look at the project root for existing `Dockerfile` or `Dockerfile.*` files.
If none exist, ask the user (or infer from the codebase) what services need
containerising, then scaffold one per service.

### Generic multi-stage template

```dockerfile
# ---- build stage ----
FROM <base-build-image> AS builder
WORKDIR /app
COPY <dependency-files> .
RUN <install-dependencies>
COPY . .
RUN <build-command>

# ---- runtime stage ----
FROM <base-runtime-image>
WORKDIR /app
COPY --from=builder /app/<output-dir> .
EXPOSE <port>
CMD ["<entrypoint>"]
```

### Common stack starters

**Node.js / Next.js**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
EXPOSE 3000
CMD ["npm", "start"]
```

**Python (generic)**
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app .
EXPOSE 8000
CMD ["python", "-m", "<module>"]
```

**Go**
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o app .

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/app .
EXPOSE 8080
CMD ["./app"]
```

### `.dockerignore` (always include)
```
.git
.env
node_modules
__pycache__
*.pyc
dist
.DS_Store
```

---

## Step 1 — Build & Push Images

### Pattern

```bash
# Build
docker build -f <DOCKERFILE> -t <REGISTRY>/<IMAGE>:<TAG> <BUILD_CONTEXT>

# Push
docker push <REGISTRY>/<IMAGE>:<TAG>
```

### Variables to confirm with the user

| Variable | Example |
|----------|---------|
| `REGISTRY` | `registry.example.com/myorg` |
| `IMAGE` | `my-api`, `my-frontend` |
| `TAG` | `1.0.0`, `latest`, git SHA |
| `DOCKERFILE` | `Dockerfile`, `api/Dockerfile` |
| `BUILD_CONTEXT` | `.`, `./api`, `./frontend` |

### Multi-service example (bash)

```bash
REGISTRY=registry.example.com/myorg
API_TAG=1.0.0
WEB_TAG=1.0.0

docker build -f api/Dockerfile   -t $REGISTRY/my-api:$API_TAG .
docker push $REGISTRY/my-api:$API_TAG

docker build -f web/Dockerfile   -t $REGISTRY/my-web:$WEB_TAG ./web
docker push $REGISTRY/my-web:$WEB_TAG
```

### Authentication

```bash
docker login <REGISTRY>              # generic
docker login ghcr.io                 # GitHub Container Registry
aws ecr get-login-password | docker login --username AWS --password-stdin <ECR_URL>
```

### Error handling
- `unauthorized` / `denied` → re-authenticate with `docker login`
- `no such file or directory` for Dockerfile → confirm path and build context
- Large image → add `.dockerignore`, use multi-stage builds

---

## Step 2 — Deploy to Kubernetes

### Upsert secrets from `.env`

```bash
kubectl create secret generic <SECRET_NAME> \
  --from-env-file=.env \
  -n <NAMESPACE> \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Apply a manifest

```bash
kubectl apply -f <MANIFEST>          # single file
kubectl apply -f <DIRECTORY>/        # all YAML in a folder
kubectl apply -k <KUSTOMIZE_DIR>     # kustomize
```

### Minimal deployment manifest template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <APP>
  namespace: <NAMESPACE>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <APP>
  template:
    metadata:
      labels:
        app: <APP>
    spec:
      containers:
        - name: <APP>
          image: <REGISTRY>/<IMAGE>:<TAG>
          ports:
            - containerPort: <PORT>
          envFrom:
            - secretRef:
                name: <SECRET_NAME>
---
apiVersion: v1
kind: Service
metadata:
  name: <APP>
  namespace: <NAMESPACE>
spec:
  selector:
    app: <APP>
  ports:
    - port: 80
      targetPort: <PORT>
```

### Verify kubectl context (MANDATORY)

Before any `kubectl apply`, **always ask the user which cluster context to use**.
Run `kubectl config get-contexts` to list available contexts, then present them
as options via `ask_user_question`. Never assume the current context is correct.

```bash
# List available contexts
kubectl config get-contexts

# Switch to user-chosen context
kubectl config use-context <CHOSEN_CONTEXT>

# Confirm
kubectl config current-context
```

If the project has a deploy script with a `REQUIRED_CONTEXT` variable, still
confirm with the user — the variable documents intent but the active context
may have drifted.

### Post-deploy verification

```bash
kubectl get pods   -n <NAMESPACE>
kubectl get svc    -n <NAMESPACE>
kubectl describe deployment <APP> -n <NAMESPACE>

# Stream logs
kubectl logs -f deployment/<APP> -n <NAMESPACE>
kubectl logs -f deployment/<APP> -n <NAMESPACE> -c <CONTAINER>   # multi-container
```

### Error handling

| Error | Fix |
|-------|-----|
| `.env file not found` | Create `.env` at the project root |
| `Failed to apply secrets` | Check context: `kubectl config current-context` |
| `ImagePullBackOff` | Image not pushed or registry creds missing in cluster |
| `CrashLoopBackOff` | Check logs: `kubectl logs deployment/<APP> -n <NAMESPACE>` |
| Manifest rejected | Dry-run first: `kubectl apply --dry-run=client -f <MANIFEST>` |

---

## Step 3 — Write `build_deploy.bat` (if missing)

After confirming the build and deploy config, check whether `build_deploy.bat` exists at the project root:

```bat
if exist build_deploy.bat goto :skip
```

If it does **not** exist, inspect the project to fill in the template variables:

| Variable | Where to find it |
|----------|------------------|
| `REGISTRY` | Ask the user, or infer from existing manifests / CI config |
| `IMAGE` | Dockerfile name, service folder name, or project name |
| `TAG` | `package.json` version, `pyproject.toml` version, or `latest` |
| `DOCKERFILE` | Path to `Dockerfile` relative to project root |
| `BUILD_CONTEXT` | Usually `.`; use service subfolder for monorepos |
| `SECRET_NAME` | Name used in K8s `secretRef`; default `<IMAGE>-secrets` |
| `NAMESPACE` | K8s namespace from existing manifests, or `default` |
| `MANIFEST` | Path to deployment YAML (e.g. `k8s/deployment.yaml`) |
| `APP` | Deployment name from the manifest |

Then **write** `build_deploy.bat` at the project root:

```bat
@echo off
setlocal
cd /d "%~dp0"

:: ── configure ────────────────────────────────────────────────────────────────
set REGISTRY=<REGISTRY>
set IMAGE=<IMAGE>
set TAG=<TAG>
set DOCKERFILE=Dockerfile
set BUILD_CONTEXT=.
set SECRET_NAME=<SECRET_NAME>
set NAMESPACE=<NAMESPACE>
set MANIFEST=<MANIFEST>
set APP=<APP>
:: ─────────────────────────────────────────────────────────────────────────────

echo [1/4] Logging in to registry ...
docker login %REGISTRY%
if errorlevel 1 ( echo [ERROR] docker login failed. & pause & exit /b 1 )

echo [2/4] Building image ...
docker build -f %DOCKERFILE% -t %REGISTRY%/%IMAGE%:%TAG% %BUILD_CONTEXT%
if errorlevel 1 ( echo [ERROR] docker build failed. & pause & exit /b 1 )

echo [3/4] Pushing image ...
docker push %REGISTRY%/%IMAGE%:%TAG%
if errorlevel 1 ( echo [ERROR] docker push failed. & pause & exit /b 1 )

echo [4/4] Deploying to Kubernetes ...

if exist .env (
    echo   Upserting secrets from .env ...
    kubectl create secret generic %SECRET_NAME% ^
        --from-env-file=.env ^
        -n %NAMESPACE% ^
        --dry-run=client -o yaml | kubectl apply -f -
    if errorlevel 1 ( echo [ERROR] Secret upsert failed. & pause & exit /b 1 )
)

kubectl apply -f %MANIFEST%
if errorlevel 1 ( echo [ERROR] kubectl apply failed. & pause & exit /b 1 )

echo.
echo [OK] Deployed %REGISTRY%/%IMAGE%:%TAG% → %NAMESPACE%/%APP%
echo.
kubectl get pods -n %NAMESPACE%
pause
endlocal
```

> **Multi-service projects:** duplicate the build/push block for each service, each with its own `IMAGE`, `TAG`, `DOCKERFILE`, and `BUILD_CONTEXT` variables.

---

## Full Release Checklist

```
[ ] Dockerfile(s) exist for every service
[ ] .dockerignore in place
[ ] docker login <REGISTRY>
[ ] docker build + push for each service  →  no errors
[ ] Manifest image tags updated to match new versions
[ ] .env present with all required secrets
[ ] kubectl context points at the right cluster
[ ] kubectl apply (secrets + manifest)    →  no errors
[ ] kubectl get pods  →  all Running
[ ] Smoke-test the app
```
