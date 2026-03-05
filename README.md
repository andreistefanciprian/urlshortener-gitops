# GitOps Kubernetes Deployments with FluxCD

This repository contains Kubernetes manifests managed by [FluxCD](https://fluxcd.io/) for automated GitOps deployments of the [URL Shortener](https://github.com/andreistefanciprian/urlshortener) application to a [private GKE cluster](https://github.com/andreistefanciprian/urlshortener/tree/main/terraform).

## Deployed Components

### Infrastructure

- **[FluxCD](https://fluxcd.io/flux/)** - GitOps toolkit for Kubernetes
- **[Kube-Prometheus-Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)** - Monitoring and alerting
- **[Cert-Manager](https://cert-manager.io/docs/installation/helm/)** - Automatic TLS certificate management
- **[Gateway API](https://gateway-api.sigs.k8s.io/)** - Kubernetes Gateway API CRDs
- **[Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/introduction)** - Secret management integration
- **[External Secrets Operator](https://external-secrets.io/)** - Syncs secrets from GCP Secret Manager into Kubernetes
- **[External DNS](https://kubernetes-sigs.github.io/external-dns/)** - Automatic DNS record management
- **[Traefik](https://doc.traefik.io/traefik/)** - Ingress controller with GKE internal load balancer
- **[Priority Classes](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)** - Pod scheduling priority configuration

### URL Shortener Application

- **[PostgreSQL](https://github.com/bitnami/charts/tree/main/bitnami/postgresql)** - Relational database (`urls` DB, `short_links` table, credentials injected via External Secrets Operator from GCP Secret Manager)
- **[Redis](https://github.com/bitnami/charts/tree/main/bitnami/redis)** - In-memory cache (standalone, credentials injected via External Secrets Operator from GCP Secret Manager)
- **[url-gen](https://github.com/andreistefanciprian/urlshortener/tree/main/url-gen)** - gRPC backend service for URL generation (credentials injected via External Secrets Operator from GCP Secret Manager)
- **[url-read](https://github.com/andreistefanciprian/urlshortener/tree/main/url-read)** - gRPC backend service for URL reading (credentials injected via External Secrets Operator from GCP Secret Manager)
- **[api-gateway](https://github.com/andreistefanciprian/urlshortener/tree/main/api-gateway)** - gRPC API gateway, aggregates url-gen and url-read backends
- **[frontend](https://github.com/andreistefanciprian/urlshortener/tree/main/frontend)** - Web frontend, communicates with api-gateway

## Initial Setup

### Prerequisites

- GitHub App for Flux authentication ([setup guide](https://fluxcd.io/blog/2025/04/flux-operator-github-app-bootstrap/#github-app-docs))
- GKE cluster provisioned via [terraform](https://github.com/andreistefanciprian/terraform-kubernetes-gke-cluster)
- `kubectl` configured to access your cluster
- `helmfile` installed locally

### Installation Steps

#### 1. Create GitHub App Secret

Follow the [Flux Operator GitHub App docs](https://fluxcd.io/blog/2025/04/flux-operator-github-app-bootstrap/#github-app-docs) to create a GitHub App, then create the Kubernetes secret:

```bash
kubectl create namespace flux-system
flux create secret githubapp flux-system \
  --app-id=<app_id> \
  --app-installation-id=<app_install_id> \
  --app-private-key=<private_key.pem>
```

#### 2. Update Configuration

Update the `GCP_PROJECT` variable in:
- `clusters/home/flux-system/cluster-vars.yaml`
- `clusters/home/flux-system/values-flux-instance.yaml`

> **Note:** Variables in the ConfigMap are propagated across all manifests in the `./infra` folder.

#### 3. Upload Secrets to GCP Secret Manager

The following secrets are **not stored in Git**. They are created automatically by [External Secrets Operator](https://external-secrets.io/), which pulls values from GCP Secret Manager:

| Secret | Namespace | GCP Secret Manager key(s) |
|--------|-----------|---------------------------|
| `cloudflare-api-token` | `external-dns`, `cert-manager` | `${CLUSTER_NAME}-cloudflare-api-token` |
| `redis-password` | `redis` | `${CLUSTER_NAME}-redis-password` |
| `pg-creds` | `postgres` | `${CLUSTER_NAME}-db-admin-password`, `${CLUSTER_NAME}-db-user-password`, `${CLUSTER_NAME}-db-replication-password` |
| `pg-creds` | `url-gen` | `${CLUSTER_NAME}-db-user-password` |
| `redis-creds` | `url-gen` | `${CLUSTER_NAME}-redis-password` |
| `pg-creds` | `url-read` | `${CLUSTER_NAME}-db-user-password` |
| `redis-creds` | `url-read` | `${CLUSTER_NAME}-redis-password` |

You must manually create and upload all secrets to GCP Secret Manager before deploying:

```bash
PROJECT_NAME="home"

# Create and populate random passwords for Redis and PostgreSQL
for secret in redis-password db-admin-password db-user-password db-replication-password; do
  echo -n "$(openssl rand -base64 32)" | gcloud secrets versions add ${PROJECT_NAME}-${secret} --data-file=-
done

# Create Cloudflare API token secret with DNS Edit permission
echo -n "<your-cloudflare-api-token>" | gcloud secrets versions add ${PROJECT_NAME}-cloudflare-api-token --data-file=-
```

> **Note:** The secrets must already exist in GCP Secret Manager before running `gcloud secrets versions add`.

#### 4. Deploy Flux Operator and Instance

```bash
# Authenticate to GHCR to be able to pull flux operator images
helm registry login ghcr.io --username <your-github-username> --password <github-PAT>

# Preview changes
helmfile -f clusters/home/flux-system/helmfile.yaml diff -l name=flux-operator

# Deploy operator
helmfile -f clusters/home/flux-system/helmfile.yaml apply -l name=flux-operator

# Deploy instance
# Preview changes
helmfile -f clusters/home/flux-system/helmfile.yaml diff -l name=flux-instance

# Deploy instance
helmfile -f clusters/home/flux-system/helmfile.yaml apply -l name=flux-instance
```

#### 5. Access Flux Operator UI

The Flux Operator UI is exposed via a private Cloudflare Tunnel ingress and is accessible at:

**https://flux-operator.9tzy.xyz/**

> **Note:** This is a private ingress. You must have [Cloudflare WARP](https://developers.cloudflare.com/cloudflare-one/connections/connect-devices/warp/) installed and connected to access it. See the [Cloudflare infrastructure setup](https://github.com/andreistefanciprian/urlshortener/tree/main/terraform) for details.

Alternatively, you can access it locally via port-forward:

```bash
kubectl port-forward svc/flux-operator -n flux-system 9080:9080
# Open in browser: http://localhost:9080
```


## Monitoring & Debugging

```bash
# View all Flux resources
kubectl get kustomizations -A
kubectl get helmrepositories -A
kubectl get helmreleases -A
kubectl get gitrepositories -A
kubectl get imagerepositories -A
kubectl get imageupdateautomations -A

# Check Helm releases
helm list -A
helm get manifest <release-name>

# Force reconciliation of a specific app
flux reconcile kustomization <app-name> --with-source

# Flux controller logs
kubectl -n flux-system logs -l app=helm-controller -f

# Delete application
kubectl delete kustomization <app-name> -n flux-system

# If your GitHub token expires, update the Flux secret
# Generate base64 encoded token
echo -n 'ghp_yourNewTokenHere' | base64

# Edit the secret and replace data.password with the new base64 value
kubectl edit secret flux-system -n flux-system
```
