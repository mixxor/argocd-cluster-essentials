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
4. Update ingress hosts in `values/gatus/values-basic.yaml`
5. Apply: `kubectl apply -f applicationsets/`

## Components

| Component | Version | Description |
|-----------|---------|-------------|
| cert-manager | v1.19.2 | TLS certificate management with Let's Encrypt |
| metrics-server | 3.12.2 | Resource metrics for HPA/VPA and `kubectl top` |
| traefik | 38.0.2 | Ingress controller (LoadBalancer for Infomaniak/Octavia) |
| gatus | 1.4.4 | Automated service health dashboard |

## Sync Waves

| Wave | Components |
|------|------------|
| 0 | cert-manager, metrics-server |
| 1 | traefik |
| 2 | gatus |

## Development

```bash
pre-commit install
```
