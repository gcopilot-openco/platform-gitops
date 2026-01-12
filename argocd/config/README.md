# ArgoCD Configuration for Crossplane Integration

This directory contains ArgoCD configuration patches required for proper Crossplane integration.

## Files

### `argocd-cm-patch.yaml`
ConfigMap patch that configures:
- **Resource tracking**: Annotation-based (required for Crossplane)
- **Resource exclusions**: Excludes ProviderConfigUsage from UI
- **Health checks**: Custom Lua scripts for platform.openco.tech resources

## Applied Configurations

### 1. ConfigMap Patch
The `argocd-cm-patch.yaml` is applied to the `argocd-cm` ConfigMap:

```bash
kubectl patch configmap argocd-cm -n argocd --type=merge --patch-file=argocd/config/argocd-cm-patch.yaml
```

After applying, restart ArgoCD components:
```bash
kubectl rollout restart statefulset argocd-application-controller -n argocd
kubectl rollout restart deployment argocd-server -n argocd
```

### 2. Kubernetes Client QPS (Applied Manually)
For improved performance with multiple CRDs, the QPS limit has been increased:

```bash
kubectl patch statefulset argocd-application-controller -n argocd --type=json \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/env/-", "value": {"name": "ARGOCD_K8S_CLIENT_QPS", "value": "300"}}]'
```

**Note**: This setting is applied via kubectl patch and won't persist through ArgoCD Helm upgrades.
To make it permanent, add to Helm values:

```yaml
controller:
  env:
    - name: ARGOCD_K8S_CLIENT_QPS
      value: "300"
```

## Health Check Logic

### AppEnvironment
- **Progressing**: Resource is being provisioned
- **Healthy**: Both `Synced=True` AND `Ready=True`
- **Degraded**: Either `Synced=False` OR `Ready=False` with error message

### GCPProject and Other Custom Resources
Similar logic checking Synced and Ready conditions from Crossplane status.

## References

- [Official Crossplane + ArgoCD Guide](https://docs.crossplane.io/latest/guides/crossplane-with-argo-cd/)
- [ArgoCD Resource Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
- [Crossplane Status Propagation](https://docs.crossplane.io/latest/concepts/composite-resources/#status)
