# ArgoCD Cluster Essentials

ApplicationSets for essential Kubernetes cluster components. Opinionated, minimal, and ready to use.

## Structure

```
applicationsets/       # ApplicationSet manifests
values/                # Helm values files
manifests/             # Additional Kubernetes manifests (ClusterIssuers, etc.)
```

## Usage

1. Fork this repository
2. Update `repoURL` in ApplicationSet files to point to your fork
3. Update email in `manifests/cert-manager/clusterissuer-*.yaml`
4. Update ingress hosts in values files to match your domain
5. Apply: `kubectl apply -f applicationsets/`

## Components

| Component | Chart Version | App Version | Description |
|-----------|---------------|-------------|-------------|
| argocd | 9.3.1 | v3.2.4 | GitOps continuous delivery |
| cert-manager | 1.17.2 | v1.17.2 | TLS certificate management with Let's Encrypt |
| gatus | 1.4.4 | v5.34.0 | Automated service health dashboard |
| metrics-server | 3.12.2 | 0.7.2 | Resource metrics for HPA/VPA and `kubectl top` |
| traefik | 38.0.2 | v3.4.0 | Ingress controller |
| vault | 0.32.0 | 1.21.2 | Secrets management |

## Ingress Endpoints

All services are exposed via Traefik ingress with TLS certificates from Let's Encrypt:

| Service | URL |
|---------|-----|
| ArgoCD | `argocd.infomaniak.moebi.io` |
| Gatus | `status.infomaniak.moebi.io` |
| Vault | `vault.infomaniak.moebi.io` |

## Sync Waves

| Wave | Components |
|------|------------|
| 0 | argocd, cert-manager, metrics-server |
| 1 | traefik |
| 2 | gatus, vault |

## ArgoCD Bootstrap

ArgoCD must be installed initially via another method (Terraform/OpenTofu, Helm, or manual install) before it can manage itself via the ApplicationSet.

**Initial install via Helm:**
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --set configs.params."server\.insecure"=true
```

**Or via OpenTofu/Terraform:**
```hcl
resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  namespace        = "argocd"
  create_namespace = true
}
```

Once ArgoCD is running, apply the ApplicationSets and ArgoCD will adopt and manage itself going forward.

## Vault Post-Install

Vault requires initialization and unsealing after deployment:

```bash
# Initialize Vault (save the unseal keys and root token!)
kubectl exec -n vault vault-0 -- vault operator init

# Unseal Vault (repeat with 3 different unseal keys)
kubectl exec -n vault vault-0 -- vault operator unseal <key>
```

## Development

```bash
pre-commit install
```
