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

The health check configuration follows the [official Crossplane + ArgoCD integration guide](https://docs.crossplane.io/latest/guides/crossplane-with-argo-cd/).

### Custom Platform Resources (platform.openco.tech/*)

#### AppEnvironment
- **Progressing**: Resource is being provisioned, waiting for conditions
- **Healthy**: Both `Synced=True` AND `Ready=True`
- **Degraded**: Either `Synced=False` OR `Ready=False` with error message

#### GCPProject and Other Composite Resources
Similar logic checking Synced and Ready conditions from Crossplane status.

### Core Crossplane Resources (*.crossplane.io/*)

Health assessment based on official Crossplane recommendations:

#### CompositeResourceDefinitions (XRDs)
- **Healthy**: When `Established=True` condition is present
- **Progressing**: No status or conditions not yet set
- **Degraded**: If `Synced=False` (configuration error)

**Note**: XRDs only have `Established` conditions, not `Synced` or `Ready`. The health check correctly handles this by accepting any of: `Ready`, `Offered`, `Established`, `ValidPipeline`, or `RevisionHealthy` as indicators of health.

#### Compositions and Configuration Resources
Resources without status (Composition, CompositionRevision, ProviderConfig, etc.) are marked as **Healthy** immediately since they are template definitions.

#### Managed Resources
- **Healthy**: When any of `Ready`, `Offered`, or `Established` = `True`
- **Degraded**: If `Synced=False` or `LastAsyncOperation=False`
- **Progressing**: Waiting for conditions to be set

#### Provider Packages
- **Healthy**: When `Healthy=True` AND `Installed=True`
- **Degraded**: If `Healthy=False`
- **Progressing**: Installation in progress

### Status Propagation

Crossplane's `function-auto-ready` propagates child resource status to parent composites:
- Parent AppEnvironment health reflects all child resources (GCPProject, GCPArtifact, etc.)
- ArgoCD only tracks parent resources in Git, child resources are dynamically created
- This provides a single health indicator for the entire infrastructure stack

## References

- [Official Crossplane + ArgoCD Guide](https://docs.crossplane.io/latest/guides/crossplane-with-argo-cd/)
- [ArgoCD Resource Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
- [Crossplane Status Propagation](https://docs.crossplane.io/latest/concepts/composite-resources/#status)
