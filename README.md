# GitOps Kubernetes Deployments with FluxCD

This repository contains Kubernetes manifests managed by [FluxCD](https://fluxcd.io/) for automated GitOps deployments of the [URL Shortener](https://github.com/andreistefanciprian/urlshortener) application to a [private GKE cluster](https://github.com/andreistefanciprian/urlshortener/terraform).

## Deployed Components

- **[FluxCD](https://fluxcd.io/flux/)** - GitOps toolkit for Kubernetes
- **[Kube-Prometheus-Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)** - Monitoring and alerting
- **[Cert-Manager](https://cert-manager.io/docs/installation/helm/)** - Automatic TLS certificate management
- **[Gateway API](https://gateway-api.sigs.k8s.io/)** - Kubernetes Gateway API CRDs
- **[Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/introduction)** - Secret management integration
- **[External Secrets Operator](https://external-secrets.io/)** - Syncs secrets from GCP Secret Manager into Kubernetes
- **[External DNS](https://kubernetes-sigs.github.io/external-dns/)** - Automatic DNS record management
- **[Priority Classes](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)** - Pod scheduling priority configuration

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

The Kubernetes secret `cloudflare-api-token` (used by External DNS and cert-manager for Cloudflare DNS01 challenges) is **not stored in Git**. It is created automatically in both the `external-dns` and `cert-manager` namespaces by [External Secrets Operator](https://external-secrets.io/), which pulls the value from GCP Secret Manager.

You must manually upload the Cloudflare API token to GCP Secret Manager before deploying.

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


```bash
# Forward port to access the operator UI
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
