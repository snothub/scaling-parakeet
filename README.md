# Kubernetes Deployment with Helm and ArgoCD

This directory contains Kubernetes infrastructure for deploying the Music Man application using Helm and ArgoCD.

## Directory Structure

```
kubernetes/
├── helm-chart/              # Helm chart for the application
│   ├── Chart.yaml          # Chart metadata
│   ├── values.yaml         # Default values
│   └── templates/          # Kubernetes manifests as templates
│       ├── namespace.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── database-statefulset.yaml
│       ├── database-service.yaml
│       └── ingress.yaml
└── argocd/                  # ArgoCD applications
    ├── application.yaml     # Dev/staging application
    └── application-prod.yaml # Production application
```

## Architecture

The deployment consists of three main components:

1. **Frontend** - React app (nginx) - 2 replicas
2. **Backend API** - Express.js with Prisma - 2 replicas
3. **Database** - PostgreSQL 16 - 1 replica (StatefulSet)

## Prerequisites

### Required Tools

- **kubectl** - Kubernetes CLI
- **helm** - Helm CLI (v3+)
- **argocd** CLI (optional but recommended)

### Kubernetes Cluster Requirements

- Kubernetes 1.19+
- Storage class for persistent volumes
- (Optional) Ingress controller (nginx, traefik, etc.)
- (Optional) cert-manager for TLS certificates
- ArgoCD installed in the cluster

## Installation Methods

### Method 1: Direct Helm Installation

**Step 1: Update values.yaml**

Edit `helm-chart/values.yaml` and update:
- Docker Hub username in image repositories
- Database password in `secrets.databasePassword`
- Other environment-specific values

**Step 2: Install with Helm**

```bash
# Install from local chart
helm install music-man ./helm-chart \
  --namespace music-man \
  --create-namespace \
  --set secrets.databasePassword="your-secure-password"

# Or with custom values file
helm install music-man ./helm-chart \
  --namespace music-man \
  --create-namespace \
  --values your-values.yaml
```

**Step 3: Verify deployment**

```bash
# Check all resources
kubectl get all -n music-man

# Check pods are running
kubectl get pods -n music-man

# Watch pod status
kubectl get pods -n music-man -w
```

### Method 2: ArgoCD (GitOps) - Recommended

**Step 1: Install ArgoCD**

```bash
# Create argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/argocd-server -n argocd

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI (or setup ingress)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access ArgoCD UI at https://localhost:8080
# Username: admin
# Password: (from above command)
```

**Step 2: Update ArgoCD Application**

Edit `argocd/application.yaml`:
- Replace `<your-username>` with your GitHub username
- Replace `<your-repo>` with your repository name
- Replace `<your-dockerhub-username>` with your Docker Hub username
- Update `secrets.databasePassword` (or use secret management)

**Step 3: Deploy via ArgoCD**

```bash
# Apply the application
kubectl apply -f argocd/application.yaml

# Check application status
argocd app get music-man

# Sync the application (if not auto-syncing)
argocd app sync music-man

# Watch sync progress
argocd app wait music-man
```

**Step 4: Access ArgoCD UI**

Visit https://localhost:8080 (or your ArgoCD ingress URL)
- View application status
- See resource tree
- Check sync status
- View logs

## Configuration

### Docker Images

Update in `values.yaml` or override:

```yaml
frontend:
  image:
    repository: yourusername/music-man
    tag: latest  # Use specific version in production

backend:
  image:
    repository: yourusername/music-man-api
    tag: latest  # Use specific version in production
```

### Frontend Configuration

The frontend uses **runtime configuration** instead of build-time environment variables. This allows the same Docker image to be used across different environments (dev, staging, production).

**How it works:**
1. The frontend Docker image includes a `/docker-entrypoint.sh` script
2. When the container starts, the script reads environment variables (`VITE_API_URL`, `VITE_SPOTIFY_CLIENT_ID`)
3. It generates a `/config.json` file with these values
4. The frontend app fetches this config at runtime

**Configuration in values.yaml:**
```yaml
frontend:
  env:
    spotifyClientId: "011c5f27eef64dd0b6f65ca673215a58"
    apiUrl: "http://backend:4000"  # Kubernetes service URL
```

**For production with ingress enabled:**
```yaml
frontend:
  env:
    spotifyClientId: "011c5f27eef64dd0b6f65ca673215a58"
    apiUrl: "/api"  # Use relative path when same domain
```

**Important:** The frontend environment variables are passed to the container at runtime via ConfigMap, not baked into the Docker image. This means you can:
- Use the same Docker image in multiple environments
- Update configuration without rebuilding the image
- Configure per-environment API URLs

### Secrets Management

**Development:**
```bash
helm install music-man ./helm-chart \
  --set secrets.databasePassword="dev-password"
```

**Production - Option 1: Sealed Secrets**
```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Create sealed secret
echo -n "your-password" | kubectl create secret generic db-password \
  --dry-run=client --from-file=password=/dev/stdin -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Apply sealed secret
kubectl apply -f sealed-secret.yaml
```

**Production - Option 2: External Secrets Operator**
```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace

# Configure secret store (AWS Secrets Manager, Vault, etc.)
# See: https://external-secrets.io/
```

### Ingress Configuration

**Enable ingress in values.yaml:**

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: music-man.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
          backend: frontend
        - path: /api
          pathType: Prefix
          backend: backend
  tls:
    - secretName: music-man-tls
      hosts:
        - music-man.yourdomain.com
```

### Resource Limits

Adjust resources in `values.yaml`:

```yaml
backend:
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
```

### Database Persistence

Configure storage:

```yaml
database:
  persistence:
    enabled: true
    storageClass: "fast-ssd"  # Your storage class
    size: 50Gi
```

## Operations

### Scaling

```bash
# Scale frontend
kubectl scale deployment frontend -n music-man --replicas=5

# Scale backend
kubectl scale deployment backend -n music-man --replicas=3

# Or update values.yaml and upgrade:
helm upgrade music-man ./helm-chart -n music-man \
  --set frontend.replicaCount=5 \
  --set backend.replicaCount=3
```

### Updating

**With Helm:**
```bash
helm upgrade music-man ./helm-chart -n music-man \
  --values updated-values.yaml
```

**With ArgoCD:**
```bash
# ArgoCD auto-syncs on git push
git add kubernetes/
git commit -m "Update deployment"
git push

# Or manual sync:
argocd app sync music-man
```

### Rollback

**With Helm:**
```bash
# List releases
helm history music-man -n music-man

# Rollback to previous
helm rollback music-man -n music-man

# Rollback to specific revision
helm rollback music-man 2 -n music-man
```

**With ArgoCD:**
```bash
# Rollback via UI or CLI
argocd app rollback music-man
```

### Monitoring

```bash
# Watch pods
kubectl get pods -n music-man -w

# Check logs
kubectl logs -f deployment/backend -n music-man
kubectl logs -f deployment/frontend -n music-man
kubectl logs -f statefulset/postgres -n music-man

# Check events
kubectl get events -n music-man --sort-by='.lastTimestamp'

# Describe resources
kubectl describe deployment backend -n music-man
```

### Database Operations

**Backup database:**
```bash
kubectl exec -it postgres-0 -n music-man -- \
  pg_dump -U spotify_user spotify_db > backup.sql
```

**Restore database:**
```bash
cat backup.sql | kubectl exec -i postgres-0 -n music-man -- \
  psql -U spotify_user spotify_db
```

**Access database:**
```bash
kubectl exec -it postgres-0 -n music-man -- \
  psql -U spotify_user -d spotify_db
```

## Troubleshooting

### Pods not starting

```bash
# Check pod status
kubectl get pods -n music-man

# Check pod events
kubectl describe pod <pod-name> -n music-man

# Check logs
kubectl logs <pod-name> -n music-man

# Check previous logs if pod restarted
kubectl logs <pod-name> -n music-man --previous
```

### Database connection issues

```bash
# Check database is ready
kubectl exec -it postgres-0 -n music-man -- pg_isready

# Check backend can reach database
kubectl exec -it deployment/backend -n music-man -- \
  nc -zv postgres 5432

# Check DATABASE_URL secret
kubectl get secret music-man-secrets -n music-man -o yaml
```

### ArgoCD sync issues

```bash
# Check application status
argocd app get music-man

# See detailed sync result
argocd app get music-man --show-operation

# Force sync
argocd app sync music-man --force

# Refresh resources
argocd app refresh music-man
```

### Image pull errors

```bash
# Check if images exist
docker pull <your-username>/music-man:latest
docker pull <your-username>/music-man-api:latest

# Create image pull secret if using private registry
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<password> \
  -n music-man
```

## Production Checklist

- [ ] Use specific image tags (not `latest`)
- [ ] Set production database password via secret management
- [ ] Configure persistent storage with appropriate storage class
- [ ] Set resource requests and limits
- [ ] Enable and configure ingress with TLS
- [ ] Set up monitoring (Prometheus, Grafana)
- [ ] Configure backup strategy for database
- [ ] Set up log aggregation (ELK, Loki)
- [ ] Configure horizontal pod autoscaling (HPA)
- [ ] Set up network policies for security
- [ ] Configure pod disruption budgets
- [ ] Enable pod security policies/standards
- [ ] Set up alerts for critical metrics

## Uninstallation

**With Helm:**
```bash
helm uninstall music-man -n music-man
kubectl delete namespace music-man
```

**With ArgoCD:**
```bash
# Delete application (with cascade deletion)
kubectl delete -f argocd/application.yaml

# Or via CLI
argocd app delete music-man --cascade
```

## Deploying New Versions

This project uses **semantic versioning** and **automated GitOps** for deployments.

### Creating a Release

Simply create and push a version tag:

```bash
git tag v1.0.1
git push origin v1.0.1
```

GitHub Actions will:
1. Build Docker images with version `v1.0.1`
2. Push to Docker Hub
3. Update ArgoCD manifests in Git
4. ArgoCD auto-syncs and deploys the new version

**See [RELEASE_PROCESS.md](../RELEASE_PROCESS.md) for complete documentation.**

### Alternative Methods

For detailed information about other deployment strategies, see:

**[ARGOCD_IMAGE_UPDATES.md](../ARGOCD_IMAGE_UPDATES.md)** - Other options include:
- Manual sync
- ArgoCD Image Updater
- Webhook triggers

**Quick Manual Sync:**
```bash
argocd app sync music-man --force
```

## References

- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [PostgreSQL on Kubernetes](https://www.postgresql.org/docs/current/index.html)
