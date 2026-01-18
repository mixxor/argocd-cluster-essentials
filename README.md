# ArgoCD Cluster Essentials

GitOps-managed Kubernetes cluster bootstrap with ArgoCD ApplicationSets. Deploys identity management, secrets, ingress, TLS, databases, and monitoring.

## Structure

```
applicationsets/       # ArgoCD ApplicationSet manifests
values/                # Helm values overrides
manifests/             # CRDs, operators, instances
```

## Components

| Component | Version | Description |
|-----------|---------|-------------|
| argocd | 9.3.4 | GitOps continuous delivery |
| cert-manager | 1.19.2 | TLS certificates via Let's Encrypt |
| cnpg | 0.27.0 | CloudNativePG PostgreSQL operator |
| external-secrets | 1.2.1 | Sync secrets from Vault to K8s |
| gatus | 1.4.4 | Service health monitoring |
| keycloak-operator | 26.5.1 | Official Keycloak Operator |
| keycloak-config-operator | 1.31.1 | EPAM operator for realms/clients via CRDs |
| metrics-server | 3.13.0 | Resource metrics for HPA/VPA |
| traefik | 38.0.2 | Ingress controller |
| vault | 0.32.0 | Secrets management |

## Sync Waves

| Wave | Components |
|------|------------|
| 0 | argocd, cert-manager, metrics-server |
| 1 | cnpg, keycloak-operator, keycloak-config-operator, traefik, external-secrets |
| 2 | gatus, vault, external-secrets-config, argocd-config |
| 4 | keycloak-instance (includes keycloak-db) |

## Ingress Endpoints

| Service | URL |
|---------|-----|
| ArgoCD | `argocd.infomaniak.moebi.io` |
| Keycloak | `keycloak.infomaniak.moebi.io` |
| Vault | `vault.infomaniak.moebi.io` |
| Gatus | `status.infomaniak.moebi.io` |

## Usage

1. Fork this repository
2. Update `repoURL` in ApplicationSet files to point to your fork
3. Update email in `manifests/cert-manager/clusterissuer-*.yaml`
4. Update ingress hosts in values files to match your domain
5. Bootstrap ArgoCD, then apply: `kubectl apply -f applicationsets/`

## ArgoCD Bootstrap

ArgoCD must be installed first before it can manage itself:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --set configs.params."server\.insecure"=true
```

## CNPG (CloudNativePG)

PostgreSQL clusters defined in `manifests/cnpg-clusters/`. Credentials auto-generated with pattern `<cluster-name>-app`.

**Keycloak Database:** `keycloak-db` (2 instances)

```bash
kubectl get secret keycloak-db-app -n keycloak -o jsonpath='{.data.password}' | base64 -d
```

## Keycloak

Two operators work together:
- **keycloak-operator**: Deploys Keycloak instances
- **keycloak-config-operator**: Manages realms, clients, users via CRDs

Admin credentials:
```bash
kubectl get secret keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d
```

### Configuring GitHub OAuth for Your Organization

This setup uses GitHub as an identity provider. To configure for your own organization:

#### 1. Create GitHub OAuth App

Go to `https://github.com/organizations/<YOUR_ORG>/settings/applications` and create a new OAuth App:

| Field | Value |
|-------|-------|
| Application name | Keycloak |
| Homepage URL | `https://keycloak.<YOUR_DOMAIN>` |
| Authorization callback URL | `https://keycloak.<YOUR_DOMAIN>/realms/<YOUR_REALM>/broker/github/endpoint` |

Save the **Client ID** and **Client Secret**.

#### 2. Store Credentials in Vault

```bash
vault kv put kv/keycloak/github-oauth \
  clientId="<GITHUB_CLIENT_ID>" \
  clientSecret="<GITHUB_CLIENT_SECRET>"
```

#### 3. Update Manifests

**manifests/keycloak/03-keycloak-realm.yaml** - Update realm name:
```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakRealm
metadata:
  name: <your-realm>
  namespace: keycloak
spec:
  realmName: <your-realm>
  keycloakRef:
    name: keycloak
    kind: Keycloak
```

**manifests/keycloak/05-github-identity-provider.yaml** - Update realm reference:
```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakRealmIdentityProvider
metadata:
  name: github
  namespace: keycloak
spec:
  realmRef:
    name: <your-realm>
    kind: KeycloakRealm
  alias: github
  providerId: github
  displayName: Sign in with GitHub
  enabled: true
  trustEmail: true
  storeToken: true
  firstBrokerLoginFlowAlias: first broker login
  config:
    clientId: $github-oauth-credentials:clientId
    clientSecret: $github-oauth-credentials:clientSecret
    defaultScope: user:email read:org
    syncMode: IMPORT
```

**manifests/keycloak/06-github-members-group.yaml** - Create authorization group:
```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakRealmGroup
metadata:
  name: <your-org>-members
  namespace: keycloak
spec:
  name: <your-org>-members
  realmRef:
    name: <your-realm>
    kind: KeycloakRealm
```

#### 4. Update ArgoCD Client (if using OIDC)

**manifests/keycloak/08-argocd-client.yaml** - Update URLs and realm:
```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakClient
metadata:
  name: argocd
  namespace: keycloak
spec:
  realmRef:
    name: <your-realm>
    kind: KeycloakRealm
  clientId: argocd
  webUrl: https://argocd.<YOUR_DOMAIN>
  redirectUris: [https://argocd.<YOUR_DOMAIN>/auth/callback]
  # ... rest of config
```

**values/argocd/values-basic.yaml** - Update OIDC issuer and group:
```yaml
configs:
  cm:
    url: https://argocd.<YOUR_DOMAIN>
    oidc.config: |
      name: Keycloak
      issuer: https://keycloak.<YOUR_DOMAIN>/realms/<your-realm>
      clientID: argocd
      clientSecret: $argocd-oidc-secret:oidc.keycloak.clientSecret
      requestedScopes: ["openid", "profile", "email", "groups"]
  rbac:
    policy.csv: |
      g, <your-org>-members, role:admin
```

#### 5. Access Control

Keycloak doesn't automatically restrict by GitHub organization. Users authenticate via GitHub, then must be added to the `<your-org>-members` group in Keycloak to access applications. Options:
- Manually assign users to the group in Keycloak admin console
- Use the "first broker login" flow to require admin approval for new users

## Vault

Initialize and unseal after deployment:

```bash
kubectl exec -n vault vault-0 -- vault operator init
kubectl exec -n vault vault-0 -- vault operator unseal <key>  # repeat 3x
```

## External Secrets

Syncs secrets from Vault to Kubernetes. Configure:
1. ClusterSecretStore in `manifests/external-secrets/`
2. ExternalSecret resources reference Vault paths

## Development

```bash
pre-commit install
```
