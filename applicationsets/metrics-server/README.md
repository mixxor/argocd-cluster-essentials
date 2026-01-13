# Metrics Server

Kubernetes Metrics Server for resource metrics collection (CPU/memory) used by HPA, VPA, and `kubectl top`.

## Deployment

This ApplicationSet deploys metrics-server to **all clusters** managed by ArgoCD automatically.

## Configuration

Update `repoURL` in `applicationset.yaml` to point to your fork of this repository.

### Values

The default values in `applications/metrics-server/values-basic.yaml` include:

- **High availability**: 2 replicas with PodDisruptionBudget
- **Security hardening**: Non-root user, read-only filesystem, dropped capabilities
- **Resource limits**: Defined CPU/memory requests and limits

### Common Customizations

For clusters with self-signed kubelet certificates, the default config includes `--kubelet-insecure-tls`. Remove this arg in production environments with proper certificates.

## Verification

After deployment, verify metrics-server is working:

```bash
kubectl top nodes
kubectl top pods -A
```
