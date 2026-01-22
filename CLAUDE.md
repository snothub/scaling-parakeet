# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains Kubernetes infrastructure for the **Music Man** application (a Spotify music player with loop functionality and lyrics display). It uses Helm for package management and ArgoCD for GitOps-based continuous deployment.

**Repository:** https://github.com/snothub/scaling-parakeet

## Architecture

The infrastructure deploys three main components:

1. **Unified App** - Single containerized deployment of both frontend (Vite/React) and backend (Express.js) running on port 4000
   - Image: `danijels/music-man` (contains both frontend and backend)
   - Default: 1 replica (configurable via `replicaCount`)

2. **PostgreSQL Database** - Persistent database for application data
   - Image: `postgres:16-alpine`
   - Database: `spotify_db` (user: `spotify_user`)
   - Persistence: Enabled by default (10Gi, configurable)

3. **Ingress** - Optional nginx-based routing (disabled by default)

**Deployment Methods:**
- **Direct Helm installation:** Manual `helm install` commands
- **ArgoCD (Recommended):** GitOps-based automatic sync from Git

## Directory Structure

```
infra/
├── helm-chart/                    # Helm chart for Kubernetes deployment
│   ├── Chart.yaml               # Chart metadata
│   ├── values.yaml              # Default configuration values
│   └── templates/               # Kubernetes manifest templates
│       ├── app-deployment.yaml
│       ├── app-service.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       ├── database-statefulset.yaml
│       ├── database-service.yaml
│       ├── ingress.yaml
│       └── namespace.yaml
├── argocd/                       # ArgoCD application manifests
│   ├── application.yaml          # Main app-of-apps for Music Man
│   └── (application-prod.yaml would be for production)
├── argoapp.yaml                  # Root ArgoCD app-of-apps manifest
├── values-prod.yaml.example      # Example production overrides
└── README.md                     # Comprehensive deployment guide
```

## Common Commands

### Helm Operations

```bash
# Install or upgrade deployment
helm install music-man ./helm-chart --namespace music-man --create-namespace
helm upgrade music-man ./helm-chart --namespace music-man

# Install with custom values
helm install music-man ./helm-chart \
  --set secrets.databasePassword="your-secure-password" \
  --values values-prod.yaml

# Verify deployment
helm list -n music-man
helm status music-man -n music-man
helm get values music-man -n music-man

# Uninstall
helm uninstall music-man -n music-man
```

### Kubernetes Debugging

```bash
# Check all resources
kubectl get all -n music-man

# Monitor pods
kubectl get pods -n music-man -w
kubectl describe pod <pod-name> -n music-man
kubectl logs -f deployment/app -n music-man

# Database access
kubectl exec -it statefulset/postgres -n music-man -- \
  psql -U spotify_user -d spotify_db

# Database backup/restore
kubectl exec -it postgres-0 -n music-man -- \
  pg_dump -U spotify_user spotify_db > backup.sql
```

### ArgoCD Operations

```bash
# Check application status
argocd app get music-man
argocd app get music-man --show-operation

# Sync application
argocd app sync music-man
argocd app sync music-man --force

# View ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080 (admin / <password-from-secret>)
```

## Configuration Hierarchy

Configuration is applied in this order (later overrides earlier):

1. **Chart defaults** (`helm-chart/values.yaml`)
2. **ArgoCD overrides** (`argocd/application.yaml` > `helm:values`)
3. **Environment-specific files** (`values-prod.yaml`)
4. **CLI parameters** (`--set` flags)

### Key Configuration Values

```yaml
# Application
app:
  replicaCount: 1                           # Pod replicas
  image.repository: danijels/music-man
  image.tag: v3.2.6                         # Updated by GitHub Actions on release
  env.spotifyClientId: 011c5f27eef64dd0b... # Spotify API client ID
  resources.requests: {memory: 256Mi, cpu: 200m}
  resources.limits: {memory: 512Mi, cpu: 500m}

# Database
database:
  persistence.enabled: true
  persistence.size: 10Gi                    # Storage allocation
  env.user: spotify_user
  env.database: spotify_db

# Secrets
secrets:
  databasePassword: "changeme-insecure-default"  # Override in production!
  jwtSecret: "changeme-insecure-default"        # For future use

# Ingress (optional)
ingress:
  enabled: false
  className: nginx
  hosts: [{host: music-man.example.com, paths: [{path: /, pathType: Prefix}]}]
```

## ArgoCD Configuration Details

The `argocd/application.yaml` file defines the GitOps workflow:

- **Source:** GitHub repo (`scaling-parakeet`) branch `HEAD` (main), path `helm-chart`
- **Destination:** Kubernetes cluster at `https://kubernetes.default.svc`, namespace `music-man`
- **Sync Policy:** Automated sync with pruning and self-healing enabled
- **Retry Logic:** Up to 5 retries with exponential backoff (5s-3m)
- **Image Updates:** Automatically updated to latest release tag by GitHub Actions

### Sync Options
- `CreateNamespace=true` - Auto-creates namespace if missing
- `Validate=true` - Validates manifests before applying
- `ServerSideApply=false` - Uses client-side apply
- `Replace=false` - Uses patch strategy

### Ignore Differences
Deployment replicas are ignored to prevent auto-rollback when manually scaled.

## Image Versioning Strategy

The application uses **semantic versioning** with GitHub Actions automation:

1. **Development:** Uses `latest` tag (in `values.yaml`)
2. **Release:** Create a version tag (`git tag v1.0.1 && git push origin v1.0.1`)
3. **CI/CD:** GitHub Actions automatically:
   - Builds and pushes image with version tag to Docker Hub
   - Updates `image.tag` in `argocd/application.yaml`
   - ArgoCD detects the change and auto-syncs the deployment

Always use specific version tags in production, never `latest`.

## Database Persistence

The StatefulSet uses Kubernetes PersistentVolumeClaims for data durability:

- **Storage Class:** Defaults to cluster default (configure in `values.yaml`)
- **Access Mode:** `ReadWriteOnce` (single node read/write)
- **Size:** 10Gi default (adjustable via `database.persistence.size`)
- **Mount Path:** `/var/lib/postgresql/data`

For production, ensure your cluster has appropriate storage provisioners configured.

## Health Checks

Both app and database include probes:

**App:**
- Liveness: Checks `/health` endpoint (30s delay, 10s interval)
- Readiness: Checks `/` endpoint (10s delay, 5s interval)

**Database:**
- Liveness: `pg_isready` command (30s delay, 10s interval)
- Readiness: `pg_isready` command (5s delay, 5s interval)

Adjust `initialDelaySeconds` if services take longer to start.

## Frontend Configuration (Runtime)

The frontend uses runtime configuration instead of build-time environment variables. This allows:
- Reusing the same image across dev/staging/production
- Updating configuration without rebuilding

**How it works:**
1. Docker entrypoint reads `VITE_API_URL` and `VITE_SPOTIFY_CLIENT_ID` environment variables
2. Generates `/config.json` with these values
3. Frontend app fetches config at runtime

**Configuration in values:**
```yaml
app:
  env:
    spotifyClientId: "011c5f27eef64dd0b..."
    apiUrl: "http://backend:4000"  # Internal k8s service URL
```

For ingress with frontend/backend on same domain, use relative path: `apiUrl: "/api"`

## Production Checklist

- [ ] Use specific image tags (never `latest`)
- [ ] Set `secrets.databasePassword` via secret management (Sealed Secrets or External Secrets Operator)
- [ ] Configure appropriate storage class for database
- [ ] Set resource requests/limits based on expected load
- [ ] Enable and configure ingress with TLS (cert-manager)
- [ ] Set up monitoring (Prometheus/Grafana)
- [ ] Configure automated database backups
- [ ] Set up log aggregation (ELK/Loki)
- [ ] Consider horizontal pod autoscaling (HPA)
- [ ] Configure network policies for security
- [ ] Set pod disruption budgets
- [ ] Enable pod security policies/standards
- [ ] Set up alerts for critical metrics

## Troubleshooting Tips

**Pods not starting:**
- Check pod status: `kubectl describe pod <name> -n music-man`
- Check logs: `kubectl logs <pod-name> -n music-man`
- Check for image pull errors: `kubectl get events -n music-man`

**Database connection issues:**
- Verify database is ready: `kubectl exec -it postgres-0 -n music-man -- pg_isready`
- Test connectivity from app pod: `kubectl exec -it deployment/app -n music-man -- nc -zv postgres 5432`
- Check DATABASE_URL secret: `kubectl get secret music-man-secrets -n music-man -o yaml`

**ArgoCD sync issues:**
- Check app status: `argocd app get music-man --show-operation`
- Force sync: `argocd app sync music-man --force`
- Refresh: `argocd app refresh music-man`

**Image pull errors:**
- Verify image exists: `docker pull danijels/music-man:v3.2.6`
- Create pull secret if using private registry

## Important Notes

- **Values override:** The `argocd/application.yaml` includes inline Helm values that override `values.yaml`. Always check both files when understanding configuration.
- **Replica counts:** ArgoCD ignores replica changes to prevent auto-rollback when manually scaling. Scale via `kubectl scale` or update values and upgrade.
- **Namespace:** All resources are created in the `music-man` namespace by default.
- **Service discovery:** Internal services use Kubernetes DNS: `app:4000` and `postgres:5432`
- **Git sync:** ArgoCD continuously monitors the main branch. All infrastructure changes should go through Git.
