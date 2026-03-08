# Kubernetes Deployment Guide

This directory contains Kubernetes manifests to deploy Didthis on a Kubernetes cluster.

## Directory Structure

```
k8s/
├── namespace.yaml          # Kubernetes namespace (didthis)
├── postgres/
│   ├── pvc.yaml            # PersistentVolumeClaim for PostgreSQL data
│   ├── secret.yaml         # PostgreSQL credentials
│   ├── deployment.yaml     # PostgreSQL Deployment
│   └── service.yaml        # PostgreSQL ClusterIP Service
├── appserver/
│   ├── configmap.yaml      # Non-sensitive app configuration
│   ├── secret.yaml         # Sensitive app credentials
│   ├── deployment.yaml     # App server Deployment (with DB migration init container)
│   ├── service.yaml        # App server ClusterIP Service
│   └── ingress.yaml        # Ingress (nginx) — exposes the app to the internet
├── discordbot/
│   ├── secret.yaml         # Discord bot credentials
│   └── deployment.yaml     # Discord bot Deployment
└── exporter/
    ├── secret.yaml         # Exporter credentials
    └── cronjob.yaml        # Content export CronJob (runs hourly by default)
```

## Prerequisites

- A running Kubernetes cluster (1.25+)
- `kubectl` configured to target the cluster
- Container images built and pushed to a registry accessible by the cluster
  (see [Building Images](#building-images) below)
- An nginx ingress controller (or adapt `appserver/ingress.yaml` to your controller):
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
  ```
- (Optional) [cert-manager](https://cert-manager.io/) for automatic TLS certificates

## Building Images

The Dockerfiles in `dockerfiles/` use multi-stage builds with a `*_prod` target for production.
Build and push the images from the repository root:

```bash
# App server
docker build -f dockerfiles/appserver --target appserver_prod \
  -t didthis/appserver:latest .
docker push didthis/appserver:latest

# Discord bot (optional)
docker build -f dockerfiles/discordbot --target discordbot_prod \
  -t didthis/discordbot:latest .
docker push didthis/discordbot:latest

# Exporter (optional)
docker build -f dockerfiles/exporter --target exporter_prod \
  -t didthis/exporter:latest .
docker push didthis/exporter:latest
```

Replace `didthis/` with your registry prefix (e.g. `ghcr.io/your-org/`), and update the
`image:` fields in the deployment / cronjob manifests accordingly.

> **Note about build-time secrets:** The production Dockerfile bakes several environment
> variables into the image at build time (e.g. `NEXT_PUBLIC_*`). Pass them as `--build-arg`
> flags when building. Runtime-only secrets are injected via Kubernetes Secrets.

## Configuration

### 1. Edit secrets

Edit the Secret files and replace every `""` placeholder with your actual value **before**
applying them to the cluster.  The files include inline comments that explain each variable.

| File | What to configure |
|------|-------------------|
| `postgres/secret.yaml` | PostgreSQL username, password, database name |
| `appserver/secret.yaml` | Database URL, session cookie secret, Firebase credentials, third-party API keys |
| `discordbot/secret.yaml` | Discord application and bot credentials |
| `exporter/secret.yaml` | Exporter access token, Cloudinary credentials |

> **Security tip:** Never commit real credentials to source control.  
> Use a tool such as [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or
> [External Secrets Operator](https://external-secrets.io/) to manage secrets safely.

### 2. Edit the ConfigMap

`appserver/configmap.yaml` contains non-sensitive configuration such as the public API
endpoint URL.  Update `NEXT_PUBLIC_API_ENDPOINT` to match your domain before deploying.

### 3. Edit the Ingress

`appserver/ingress.yaml` is configured for the nginx ingress controller.  
Replace `didthis.example.com` with your actual domain.  
Uncomment the `tls` section and configure it if you want HTTPS.

## Deploying

Apply the manifests in the following order (dependencies first):

```bash
# 1. Create namespace
kubectl apply -f k8s/namespace.yaml

# 2. Deploy PostgreSQL
kubectl apply -f k8s/postgres/

# 3. Wait for PostgreSQL to be ready
kubectl rollout status deployment/postgres -n didthis

# 4. Deploy app server
kubectl apply -f k8s/appserver/

# 5. Wait for app server to be ready
kubectl rollout status deployment/appserver -n didthis

# 6. (Optional) Deploy Discord bot
kubectl apply -f k8s/discordbot/

# 7. (Optional) Deploy exporter CronJob
kubectl apply -f k8s/exporter/
```

### Apply everything at once

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres/ -f k8s/appserver/ -f k8s/discordbot/ -f k8s/exporter/
```

## Verifying the Deployment

```bash
# Check all resources in the namespace
kubectl get all -n didthis

# Check app server logs
kubectl logs -l app=appserver -n didthis

# Check PostgreSQL logs
kubectl logs -l app=postgres -n didthis
```

## Firebase Credentials

Firebase is used for authentication.  The app server reads the service account key from a
file mounted at `/secrets/firebase-credentials.json`, specified via the
`GOOGLE_APPLICATION_CREDENTIALS` environment variable (which is set automatically in the
Deployment manifest).

1. In the [Firebase console](https://console.firebase.google.com/), go to  
   **Project settings → Service accounts → Generate new private key**.
2. Open `appserver/secret.yaml` and paste the full JSON file contents as the value of
   the `firebase-credentials.json` key in `stringData`:
   ```yaml
   stringData:
     firebase-credentials.json: |
       {
         "type": "service_account",
         "project_id": "...",
         ...
       }
   ```
3. The Deployment mounts this value as a file at `/secrets/firebase-credentials.json`
   inside the container and sets `GOOGLE_APPLICATION_CREDENTIALS` accordingly.

## Upgrading

```bash
# Rebuild and push new image, then roll out
docker build -f dockerfiles/appserver --target appserver_prod -t didthis/appserver:v2 .
docker push didthis/appserver:v2

# Update the image in the deployment
kubectl set image deployment/appserver appserver=didthis/appserver:v2 -n didthis
kubectl rollout status deployment/appserver -n didthis
```

## Teardown

```bash
kubectl delete namespace didthis
```
