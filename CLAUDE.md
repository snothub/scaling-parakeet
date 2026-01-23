# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains Kubernetes infrastructure for the **Music Man** application (a Spotify music player with loop functionality and lyrics display). It uses Helm for package management, Traefik for ingress, cert-manager for HTTPS certificates, and ArgoCD for GitOps-based continuous deployment.

**Repository:** https://github.com/snothub/scaling-parakeet

## Architecture

The infrastructure uses a **3-tier layered approach** with ArgoCD managing all components:

### Tier 1: Cluster Infrastructure
- **Traefik v3** - Ingress controller with HTTP→HTTPS redirect
  - LoadBalancer service type (creates Azure public IP)
  - Manages all inbound traffic to the cluster
  - Deployed via `argocd/traefik-application.yaml`

- **cert-manager v1.14** - Automated HTTPS certificate management
  - Handles certificate issuance, renewal, and rotation
  - Integrates with Let's Encrypt for TLS certificates
  - Deployed via `argocd/cert-manager-application.yaml`

### Tier 2: Certificate Management
- **Let's Encrypt ClusterIssuers** - Two certificate issuers
  - `letsencrypt-staging` - For testing (higher rate limits)
  - `letsencrypt-prod` - For production (browser-trusted)
  - Configured in `cluster-resources/letsencrypt-issuer.yaml`
  - Deployed via `argocd/cluster-resources-application.yaml`

### Tier 3: Application Layer
- **Unified App** - Single containerized deployment of frontend (Vite/React) + backend (Express.js) on port 4000
  - Image: `danijels/music-man` (contains both components)
  - Default: 1 replica (configurable)
  - Health checks: liveness `/health`, readiness `/`

- **PostgreSQL Database** - Persistent data storage
  - Image: `postgres:16-alpine`
  - Database: `spotify_db` (user: `spotify_user`)
  - Persistence: 10Gi PVC with StatefulSet

- **Traefik Ingress** - Routes HTTPS traffic to application
  - Domain: `music-man.mywire.org` (or configured domain)
  - TLS termination with automatic certificate management
  - HTTP automatically redirects to HTTPS

**Deployment Methods:**
- **Direct Helm:** Manual `helm install` commands
- **ArgoCD (Recommended):** GitOps-based automatic sync from Git

## Directory Structure

```
infra/
├── helm-chart/                      # Helm chart for Music Man application
│   ├── Chart.yaml
│   ├── values.yaml                  # Default configuration values
│   └── templates/                   # Kubernetes manifest templates
│       ├── app-deployment.yaml
│       ├── app-service.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       ├── database-statefulset.yaml
│       ├── database-service.yaml
│       ├── ingress.yaml
│       └── namespace.yaml
├── argocd/                          # ArgoCD application manifests
│   ├── application.yaml             # Music Man application (Helm chart)
│   ├── traefik-application.yaml     # Traefik ingress controller
│   ├── cert-manager-application.yaml # cert-manager deployment
│   └── cluster-resources-application.yaml # Cluster resources (issuers, namespaces)
├── cluster-resources/               # Kubernetes resources managed by ArgoCD
│   ├── letsencrypt-issuer.yaml      # Let's Encrypt ClusterIssuers (staging + prod)
│   ├── cert-manager-namespace.yaml  # cert-manager namespace with Istio disabled
│   └── istio-namespace-label.yaml   # Disable Istio for music-man namespace
├── README.md                        # Comprehensive deployment guide
└── CLAUDE.md                        # This file
```

## Common Commands

### Helm Operations

```bash
# Install or upgrade Music Man application only
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

# Uninstall Music Man only
helm uninstall music-man -n music-man
```

### ArgoCD Infrastructure Deployment (Cluster-wide)

```bash
# Deploy cluster infrastructure via ArgoCD (one-time)
kubectl apply -f argocd/traefik-application.yaml
kubectl apply -f argocd/cert-manager-application.yaml
kubectl apply -f argocd/cluster-resources-application.yaml

# Or deploy the main Music Man application
kubectl apply -f argocd/application.yaml

# Check application status
argocd app get music-man
argocd app get traefik
argocd app get cert-manager
argocd app get cluster-resources

# Sync applications (if not auto-syncing)
argocd app sync music-man
argocd app sync traefik --force
argocd app sync cert-manager --force
argocd app sync cluster-resources --force

# View ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080 (admin / <password-from-secret>)
```

### Kubernetes Debugging

```bash
# Check all resources in music-man
kubectl get all -n music-man

# Monitor pods
kubectl get pods -n music-man -w
kubectl describe pod <pod-name> -n music-man
kubectl logs -f deployment/app -n music-man

# Check Traefik status
kubectl get pods -n traefik-system
kubectl get svc traefik -n traefik-system  # Get LoadBalancer IP
kubectl logs -n traefik-system -l app.kubernetes.io/name=traefik --tail=50

# Check cert-manager status
kubectl get pods -n cert-manager
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod

# Check certificate status
kubectl get certificate -n music-man
kubectl describe certificate music-man-tls -n music-man

# Database access
kubectl exec -it statefulset/postgres -n music-man -- \
  psql -U spotify_user -d spotify_db

# Database backup/restore
kubectl exec -it postgres-0 -n music-man -- \
  pg_dump -U spotify_user spotify_db > backup.sql
```

### HTTPS Certificate Management

```bash
# Monitor certificate issuance
kubectl get certificate -n music-man -w
kubectl describe certificate music-man-tls -n music-man

# Check Let's Encrypt challenges
kubectl get challenges -n music-man
kubectl describe challenge <challenge-name> -n music-man

# View certificate content
kubectl get secret music-man-tls -n music-man -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# Trigger certificate renewal (delete the certificate)
kubectl delete certificate music-man-tls -n music-man

# Switch between staging and production issuers
# Edit helm-chart/values.yaml and change:
# annotations:
#   cert-manager.io/cluster-issuer: letsencrypt-staging  # or letsencrypt-prod
```

### Ingress and DNS

```bash
# Get Traefik LoadBalancer IP
kubectl get svc traefik -n traefik-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Check ingress configuration
kubectl get ingress -n music-man
kubectl describe ingress music-man-ingress -n music-man

# Test HTTP redirect to HTTPS
curl -I http://music-man.mywire.org
# Should redirect to https://...

# Test HTTPS access
curl -I https://music-man.mywire.org
# Should return 200 OK with valid certificate
```

## Configuration Hierarchy

Configuration is applied in this order (later overrides earlier):

1. **Chart defaults** (`helm-chart/values.yaml`)
2. **ArgoCD overrides** (`argocd/application.yaml` > `helm:values`)
3. **Environment-specific files** (`values-prod.yaml`)
4. **CLI parameters** (`--set` flags)

### Key Configuration Values

#### Music Man Application (helm-chart/values.yaml)

```yaml
app:
  replicaCount: 1
  image:
    repository: danijels/music-man
    tag: v3.4.5                             # Updated by GitHub Actions on release
  env:
    port: "4000"
    spotifyClientId: "011c5f27eef64dd0b..."
  resources:
    requests: {memory: 256Mi, cpu: 200m}
    limits: {memory: 512Mi, cpu: 500m}

database:
  persistence:
    enabled: true
    size: 10Gi
  env:
    user: spotify_user
    database: spotify_db

secrets:
  databasePassword: "changeme-insecure-default"  # Override in production!
  jwtSecret: "changeme-insecure-default"

ingress:
  enabled: true
  className: traefik                        # Uses Traefik ingress controller
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging  # or letsencrypt-prod
  hosts:
    - host: music-man.mywire.org            # Configure your domain
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: music-man-tls
      hosts:
        - music-man.mywire.org
```

#### Traefik Configuration (argocd/traefik-application.yaml)

```yaml
# Traefik v3 with HTTP→HTTPS redirect
deployment:
  replicas: 2                               # High availability

service:
  type: LoadBalancer                        # Creates Azure public IP
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: /ping

entryPoints:
  web:
    address: :80
    http:
      redirections:
        entrypoint:
          to: websecure                     # HTTP redirects to HTTPS
          scheme: https
  websecure:
    address: :443
    http:
      tls: {}                               # TLS enabled on port 443
```

#### cert-manager Configuration (argocd/cert-manager-application.yaml)

```yaml
installCRDs: true                           # Install CustomResourceDefinitions

webhook:
  enabled: true
  podAnnotations:
    sidecar.istio.io/inject: "false"        # Disable Istio for webhook

caInjector:
  enabled: true
  podAnnotations:
    sidecar.istio.io/inject: "false"        # Disable Istio for CA injector

podAnnotations:
  sidecar.istio.io/inject: "false"          # Disable Istio for controller
```

#### Let's Encrypt Issuers (cluster-resources/letsencrypt-issuer.yaml)

```yaml
letsencrypt-staging:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: danijels@gmail.com
    privateKeySecretRef: letsencrypt-staging-key
    solvers:
      - http01:
          ingress:
            class: traefik

letsencrypt-prod:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: danijels@gmail.com
    privateKeySecretRef: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

## ArgoCD Configuration Details

### Music Man Application (`argocd/application.yaml`)

- **Source:** GitHub repo (`scaling-parakeet`) branch `HEAD` (main), path `helm-chart`
- **Destination:** Kubernetes cluster at `https://kubernetes.default.svc`, namespace `music-man`
- **Sync Policy:** Automated sync with pruning and self-healing enabled
- **Retry Logic:** Up to 5 retries with exponential backoff (5s-3m)
- **Image Updates:** Automatically updated to latest release tag by GitHub Actions

**Sync Options:**
- `CreateNamespace=true` - Auto-creates namespace if missing
- `Validate=true` - Validates manifests before applying
- `ServerSideApply=false` - Uses client-side apply
- `Replace=false` - Uses patch strategy

**Ignore Differences:**
- Deployment replicas are ignored to prevent auto-rollback when manually scaled

### Traefik Application (`argocd/traefik-application.yaml`)

- **Chart:** `traefik` from `https://traefik.github.io/charts`
- **Version:** v28.0.0 (Traefik v3)
- **Namespace:** `traefik-system`
- **Replicas:** 2 (high availability)
- **Service Type:** LoadBalancer (creates Azure public IP)

**Key Features:**
- HTTP port 80: Redirects to HTTPS (websecure)
- HTTPS port 443: TLS-enabled ingress
- IngressClass set as default (traefik)
- Health probe annotation for Azure Load Balancer

### cert-manager Application (`argocd/cert-manager-application.yaml`)

- **Chart:** `cert-manager` from `https://charts.jetstack.io`
- **Version:** v1.14.0
- **Namespace:** `cert-manager`
- **Replicas:** 1 (controller, webhook, cainjector)
- **CRDs:** Installed automatically via Helm

**Key Features:**
- Istio sidecar injection disabled (all components)
- Webhook enabled for validation
- CA Injector enabled for certificate rotation
- Namespace labeled `istio-injection: disabled`

### Cluster Resources Application (`argocd/cluster-resources-application.yaml`)

- **Source:** GitHub repo path `cluster-resources`
- **Destination:** Namespace `cert-manager`
- **Resources Managed:**
  - Let's Encrypt ClusterIssuers (staging + production)
  - Namespace configurations (Istio injection disabled)

**Deployment Order (Critical):**
1. Traefik (must be ready to accept traffic)
2. cert-manager (must be ready to validate certificates)
3. Cluster Resources (creates issuers for cert-manager)
4. Music Man Application (uses Traefik ingress + Let's Encrypt certificates)

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

## Traefik Ingress and HTTPS Setup

### Overview

The infrastructure uses **Traefik v3** as the ingress controller with **cert-manager** and **Let's Encrypt** for automated HTTPS certificates.

**Traffic Flow:**
1. Client requests `https://music-man.mywire.org`
2. DNS resolves to Traefik LoadBalancer IP
3. Traefik terminates HTTPS using cert-manager-issued certificate
4. Traefik routes to `app` service on port 4000
5. HTTP traffic auto-redirects to HTTPS

### Setting Up HTTPS

**Prerequisites:**
- Domain name (e.g., `music-man.mywire.org`)
- DuckDNS or similar dynamic DNS service
- Traefik deployed and LoadBalancer IP obtained

**Steps:**

1. **Get Traefik LoadBalancer IP:**
   ```bash
   kubectl get svc traefik -n traefik-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
   # Example output: 4.220.42.191
   ```

2. **Update DNS to point to LoadBalancer IP:**
   - Go to your DNS provider (e.g., DuckDNS at https://www.duckdns.org)
   - Update `music-man.duckdns.org` to IP `4.220.42.191`
   - Verify: `nslookup music-man.duckdns.org` returns your IP

3. **Update domain in values.yaml:**
   ```yaml
   ingress:
     hosts:
       - host: music-man.duckdns.org  # Your domain
         paths:
           - path: /
             pathType: Prefix
     tls:
       - secretName: music-man-tls
         hosts:
           - music-man.duckdns.org
   ```

4. **Deploy with staging issuer first:**
   ```yaml
   annotations:
     cert-manager.io/cluster-issuer: letsencrypt-staging  # For testing
   ```
   - Staging has higher rate limits and doesn't require rate limit waiting
   - Browser will show certificate warning (expected for staging)
   - Verifies HTTP-01 challenge works before production

5. **Monitor certificate issuance:**
   ```bash
   kubectl get certificate -n music-man -w
   kubectl describe certificate music-man-tls -n music-man
   kubectl get challenges -n music-man  # View active HTTP-01 challenges
   ```

6. **Switch to production (after staging works):**
   - Update `cert-manager.io/cluster-issuer: letsencrypt-prod`
   - Delete staging certificate: `kubectl delete certificate music-man-tls -n music-man`
   - Wait for production certificate to issue
   - Browser will show valid certificate (no warning)

### Troubleshooting HTTPS Issues

**DNS not resolving:**
- Verify DNS provider has correct IP
- Check with: `nslookup music-man.duckdns.org`
- Wait 5-10 minutes for DNS propagation

**Certificate stuck in "Pending":**
- Check ingress exists: `kubectl get ingress -n music-man`
- Check challenges: `kubectl get challenges -n music-man`
- Check cert-manager logs: `kubectl logs -n cert-manager -l app=cert-manager --tail=50`

**HTTP-01 challenge failing:**
- Verify HTTP endpoint is reachable: `curl -v http://music-man.duckdns.org/`
- Check Traefik is receiving traffic: `kubectl logs -n traefik-system -l app.kubernetes.io/name=traefik`
- Verify namespace has Istio injection disabled: `kubectl get namespace music-man -o jsonpath='{.metadata.labels.istio-injection}'`
  - Should return `disabled`

**"too many failed authorizations" error:**
- You've hit Let's Encrypt rate limit
- Wait 1 hour before retrying
- Use staging issuer for testing to avoid rate limits

**Certificate not renewing:**
- cert-manager auto-renews 30 days before expiry
- Check renew time: `kubectl get certificate -n music-man -o jsonpath='{.status.renewalTime}'`
- Manually trigger renewal: `kubectl delete certificate music-man-tls -n music-man`

### Istio Integration

If your cluster uses Istio service mesh:

**Important:** Istio sidecar injection must be **disabled** for:
- `cert-manager` namespace
- `traefik-system` namespace (if using Istio there)
- HTTP-01 solver pods in `music-man` namespace

This is already configured in:
- `cluster-resources/cert-manager-namespace.yaml` - Labels cert-manager namespace
- `cluster-resources/istio-namespace-label.yaml` - Labels music-man namespace
- `argocd/cert-manager-application.yaml` - Pod annotations for cert-manager pods

If Istio is re-injecting sidecars, verify namespace labels:
```bash
kubectl get namespace cert-manager -o jsonpath='{.metadata.labels.istio-injection}'
kubectl get namespace music-man -o jsonpath='{.metadata.labels.istio-injection}'
# Both should return "disabled"
```

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

### Infrastructure
- [ ] Deploy Traefik ingress controller with LoadBalancer service
- [ ] Deploy cert-manager with CRDs auto-installed
- [ ] Configure Let's Encrypt ClusterIssuers (staging for testing, prod for real)
- [ ] Disable Istio sidecar injection for cert-manager and solver namespaces
- [ ] Configure DNS to point domain to Traefik LoadBalancer IP
- [ ] Test HTTP→HTTPS redirect works
- [ ] Test HTTPS access with valid certificate

### Application
- [ ] Use specific image tags (never `latest`) - e.g., `v3.4.5`
- [ ] Enable ingress with Traefik ingressClass
- [ ] Set domain in ingress.hosts (e.g., `music-man.mywire.org`)
- [ ] Use `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation
- [ ] Set `secrets.databasePassword` via secret management (Sealed Secrets or External Secrets Operator)
- [ ] Configure appropriate storage class for database
- [ ] Set resource requests/limits based on expected load
- [ ] Review health check timeouts (`initialDelaySeconds`, `periodSeconds`)

### Monitoring & Operations
- [ ] Set up monitoring (Prometheus/Grafana)
- [ ] Configure automated database backups
- [ ] Set up log aggregation (ELK/Loki)
- [ ] Consider horizontal pod autoscaling (HPA)
- [ ] Configure network policies for security
- [ ] Set pod disruption budgets
- [ ] Enable pod security policies/standards
- [ ] Set up alerts for critical metrics
- [ ] Monitor certificate expiry: `kubectl get certificate -n music-man -o jsonpath='{.items[0].status.notAfter}'`

### Security
- [ ] Rotate database password regularly
- [ ] Use sealed secrets or external secrets operator for sensitive data
- [ ] Restrict ingress via network policies
- [ ] Enable pod security policies
- [ ] Configure RBAC for service accounts
- [ ] Scan container images for vulnerabilities

## Troubleshooting Guide

### Application Deployment Issues

**Pods not starting:**
```bash
kubectl describe pod <pod-name> -n music-man
kubectl logs <pod-name> -n music-man
kubectl get events -n music-man --sort-by='.lastTimestamp'
```

**Database connection issues:**
```bash
# Check database is ready
kubectl exec -it postgres-0 -n music-man -- pg_isready

# Test connectivity from app pod
kubectl exec -it deployment/app -n music-man -- nc -zv postgres 5432

# Check connection string
kubectl get secret music-man-secrets -n music-man -o yaml | grep DATABASE_URL
```

**Image pull errors:**
```bash
# Verify image exists
docker pull danijels/music-man:v3.4.5

# Create pull secret if using private registry
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=<username> \
  --docker-password=<password> \
  -n music-man
```

### Traefik Issues

**LoadBalancer IP not assigned:**
```bash
# Check service status
kubectl get svc traefik -n traefik-system -w

# Check for provisioning errors
kubectl describe svc traefik -n traefik-system
```

**Ingress not routing traffic:**
```bash
# Check Traefik sees the ingress
kubectl get ingress -n music-man -o yaml | kubectl exec -i $(kubectl get pods -n traefik-system -l app.kubernetes.io/name=traefik -o jsonpath='{.items[0].metadata.name}') -n traefik-system -- cat

# Check Traefik logs for routing errors
kubectl logs -n traefik-system -l app.kubernetes.io/name=traefik --tail=100 | grep -i "music-man\|error"
```

### cert-manager & HTTPS Issues

**Certificate not issuing (detailed debugging):**
```bash
# 1. Check certificate resource
kubectl describe certificate music-man-tls -n music-man

# 2. Check for certificate request
kubectl get certificaterequest -n music-man
kubectl describe certificaterequest <cr-name> -n music-man

# 3. Check for order
kubectl get order -n music-man
kubectl describe order <order-name> -n music-man

# 4. Check for challenges
kubectl get challenges -n music-man
kubectl describe challenge <challenge-name> -n music-man

# 5. Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=100
```

**HTTP-01 challenge timing out:**
```bash
# Verify DNS resolution
nslookup music-man.duckdns.org

# Test HTTP accessibility
curl -v http://music-man.duckdns.org/.well-known/acme-challenge/test

# Check solver pods
kubectl get pods -n music-man -l acme.cert-manager.io/http01-solver=true

# Check ingress routes solver traffic
kubectl get ingress -n music-man
kubectl describe ingress cm-acme-http-solver-* -n music-man 2>/dev/null || echo "No solver ingress found"

# Check Traefik routing
kubectl logs -n traefik-system -l app.kubernetes.io/name=traefik --tail=50 | grep acme
```

**Rate limit hit (too many failed authorizations):**
- You've hit Let's Encrypt rate limit on this domain
- Wait 1 hour before retrying
- Always test with staging issuer first: `cert-manager.io/cluster-issuer: letsencrypt-staging`
- Staging issuers have higher rate limits for testing

**cert-manager pods have Istio sidecars (mTLS issues):**
```bash
# Check if Istio is injecting sidecars
kubectl get pods -n cert-manager -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.containers | length}{" containers"}{"\n"}{end}'

# Should show 1 container per pod, if 2 then Istio sidecar was injected

# Fix: Label namespace to disable injection
kubectl label namespace cert-manager istio-injection=disabled --overwrite
kubectl delete pods --all -n cert-manager  # Force pod restart
```

### ArgoCD Sync Issues

**Application sync failing:**
```bash
# Check application status
argocd app get music-man
argocd app get music-man --show-operation

# See sync error details
argocd app get music-man --refresh

# Force sync (useful after fixing issues)
argocd app sync music-man --force

# Refresh to re-check Git repository
argocd app refresh music-man
```

**Helm template rendering errors:**
```bash
# Test Helm template locally
helm template music-man ./helm-chart --namespace music-man

# Test with ArgoCD values
helm template music-man ./helm-chart \
  --values <(echo "app:\n  image:\n    tag: v3.4.5")
```

## Important Notes

- **Values override:** The `argocd/application.yaml` includes inline Helm values that override `values.yaml`. Always check both files when understanding configuration.
- **Replica counts:** ArgoCD ignores replica changes to prevent auto-rollback when manually scaling. Scale via `kubectl scale` or update values and upgrade.
- **Namespace:** All resources are created in the `music-man` namespace by default.
- **Service discovery:** Internal services use Kubernetes DNS: `app:4000` and `postgres:5432`
- **Git sync:** ArgoCD continuously monitors the main branch. All infrastructure changes should go through Git.

### Deployment Strategy
- **Deployment order matters:** Deploy in this order: Traefik → cert-manager → Cluster Resources → Music Man
  - Traefik must be ready to accept traffic before cert-manager HTTP-01 challenges
  - cert-manager must be ready before ClusterIssuers can validate
  - ClusterIssuers must exist before Music Man ingress can request certificates

- **Replica counts for Music Man:** ArgoCD ignores replica counts to prevent auto-rollback when manually scaled. Scale via `kubectl scale` or update values and upgrade.

- **Certificate issuance timing:** First certificate typically takes 30-60 seconds with staging issuer, 2-5 minutes with production (includes Let's Encrypt validation).

### Namespaces & Service Discovery
- **Namespaces:** Resources spread across multiple namespaces:
  - `music-man` - Application and database
  - `traefik-system` - Traefik ingress controller
  - `cert-manager` - cert-manager and webhook
  - `argocd` - ArgoCD itself (managed separately)

- **Service discovery:** Internal services use Kubernetes DNS:
  - `app:4000` - Music Man application service
  - `postgres:5432` - PostgreSQL database service
  - `traefik.traefik-system:80` - Traefik from inside cluster

### GitOps Workflow
- **Git is source of truth:** ArgoCD continuously monitors the main branch. All infrastructure changes should go through Git
- **Avoid manual kubectl changes:** They will be overwritten by ArgoCD on next sync. Use `git add/commit/push` instead
- **Secrets management:** For production, don't commit secrets to Git. Use Sealed Secrets or External Secrets Operator

### Istio Service Mesh Compatibility
- **Sidecar injection must be disabled** for cert-manager and HTTP-01 solver pods
- If you have Istio, ensure:
  - `cert-manager` namespace: `istio-injection: disabled`
  - `music-man` namespace: `istio-injection: disabled` (for HTTP-01 solvers)
  - cert-manager pods: `sidecar.istio.io/inject: "false"` annotation
- **Why?** HTTP-01 challenges require direct cert-manager↔Traefik communication. Istio mTLS intercepts and breaks this.

### Rate Limiting & Certificate Management
- **Test with staging first:** Use `letsencrypt-staging` before production (higher rate limits)
  - Staging: 150 certs per domain per day
  - Production: 50 certs per domain per week
- **Production fails?** Wait 1 hour before retrying
- **Certificate renewal:** Auto-renews 30 days before expiry. Monitor: `kubectl get certificate -n music-man -o jsonpath='{.items[0].status.renewalTime}'`

### Image Updates
- **GitHub Actions automation:** Tag releases with `git tag v3.4.5 && git push origin v3.4.5` to trigger:
  1. Docker image build
  2. Push to Docker Hub
  3. Update `app.image.tag` in `argocd/application.yaml`
  4. ArgoCD auto-syncs deployment
- **Never use `latest` in production** - Always use specific version tags
