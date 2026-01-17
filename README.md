# ArgoCD Cluster Essentials

ApplicationSets for essential Kubernetes cluster components. Opinionated, minimal, and ready to use.

## Structure

```
applicationsets/       # ApplicationSet manifests
values/                # Helm values files
manifests/             # Additional Kubernetes manifests (CRDs, operators, instances)
```

## Usage

1. Fork this repository
2. Update `repoURL` in ApplicationSet files to point to your fork
3. Update email in `manifests/cert-manager/clusterissuer-*.yaml`
4. Update ingress hosts in values files and manifests to match your domain
5. Apply: `kubectl apply -f applicationsets/`

## Components

| Component | Chart/Version | Description |
|-----------|---------------|-------------|
| argocd | 9.3.1 | GitOps continuous delivery |
| cert-manager | 1.17.2 | TLS certificate management with Let's Encrypt |
| cnpg | 0.27.0 | CloudNativePG PostgreSQL operator |
| gatus | 1.4.4 | Automated service health dashboard |
| keycloak-operator | 26.5.1 | Official Keycloak Operator (deploys instances) |
| keycloak-config-operator | 1.31.1 | EPAM Keycloak Operator (manages realms/clients via CRDs) |
| metrics-server | 3.12.2 | Resource metrics for HPA/VPA and `kubectl top` |
| traefik | 38.0.2 | Ingress controller |
| vault | 0.32.0 | Secrets management |

## Ingress Endpoints

All services are exposed via Traefik ingress with TLS certificates from Let's Encrypt:

| Service | URL |
|---------|-----|
| ArgoCD | `argocd.infomaniak.moebi.io` |
| Gatus | `status.infomaniak.moebi.io` |
| Keycloak | `keycloak.infomaniak.moebi.io` |
| Vault | `vault.infomaniak.moebi.io` |

## Sync Waves

| Wave | Components |
|------|------------|
| 0 | argocd, cert-manager, metrics-server |
| 1 | cnpg, keycloak-operator, keycloak-config-operator, traefik |
| 2 | cnpg-clusters, gatus, vault |
| 3 | keycloak-instance |

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

## CNPG (CloudNativePG)

PostgreSQL databases are managed by CloudNativePG operator. Database clusters are defined in `manifests/cnpg-clusters/`.

CNPG auto-generates credentials secrets with the naming pattern `<cluster-name>-app`.

**Keycloak Database:**
- Cluster: `keycloak-db` (2 instances)
- Database: `keycloak`
- Auto-generated secret: `keycloak-db-app`

View the generated password:
```bash
kubectl get secret keycloak-db-app -n keycloak -o jsonpath='{.data.password}' | base64 -d
```

## Keycloak (Operator-based)

Keycloak is deployed using two operators:

1. **Official Keycloak Operator** (`keycloak-operator`) - Deploys and manages Keycloak instances
2. **EPAM Keycloak Config Operator** (`keycloak-config-operator`) - Manages realms, clients, users via CRDs

### Architecture

```
keycloak-operator (Official)     keycloak-config-operator (EPAM)
        │                                    │
        ▼                                    ▼
   Keycloak CR ──────────────────────► Keycloak CR (connection)
   (k8s.keycloak.org/v2alpha1)         (v1.edp.epam.com/v1)
        │                                    │
        ▼                                    ▼
   Keycloak Instance              KeycloakRealm, KeycloakClient,
                                  KeycloakRealmUser, etc.
```

### Managing Tenants via GitOps

Create realms and clients declaratively:

```yaml
# manifests/keycloak/realms/my-tenant.yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakRealm
metadata:
  name: my-tenant
  namespace: keycloak
spec:
  realmName: my-tenant
  keycloakRef:
    name: keycloak
    kind: Keycloak
---
apiVersion: v1.edp.epam.com/v1
kind: KeycloakClient
metadata:
  name: my-app
  namespace: keycloak
spec:
  clientId: my-app
  public: true
  realmRef:
    name: my-tenant
    kind: KeycloakRealm
```

### Initial Admin Credentials

The Keycloak Operator auto-generates initial admin credentials:

```bash
# Get the initial admin password
kubectl get secret keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d
```

Default admin user is `admin`. Access the admin console at `https://keycloak.infomaniak.moebi.io/admin`.

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
