# ArgoCD Cluster Essentials

ApplicationSets for essential Kubernetes cluster components. Opinionated, minimal, and ready to use.

## Structure

```
applications/          # Helm values files
cluster-essentials/    # ApplicationSet manifests
```

## Usage

1. Fork this repository
2. Update `repoURL` in ApplicationSet files to point to your fork
3. Apply: `kubectl apply -f cluster-essentials/<app>/applicationset.yaml`

## Components

| Component | Description |
|-----------|-------------|
| metrics-server | Resource metrics for HPA/VPA and `kubectl top` |

## Development

```bash
pre-commit install
```
