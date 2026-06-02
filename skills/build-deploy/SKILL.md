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
