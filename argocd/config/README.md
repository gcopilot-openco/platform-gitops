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

### Philosophy: Detailed Health Checks for XRs, Built-in for Infrastructure

We only define custom health checks for our **composite resources (XRs)** at `platform.openco.tech/*`. For Crossplane infrastructure components (Providers, XRDs, Compositions), we rely on ArgoCD's built-in health checks where available.

### Custom Platform Resources (platform.openco.tech/*)

These are our **business-level abstractions** that need detailed health visibility:

#### AppEnvironment (Top-level Composite)
- **Progressing**: Resource is being provisioned, waiting for conditions
- **Healthy**: Both `Synced=True` AND `Ready=True`
- **Degraded**: Either `Synced=False` OR `Ready=False` with error message

#### GCPProject, GCPArtifact, GCPSecretManager, GCPCloudRun (Child Composites)
Similar logic checking Synced and Ready conditions from Crossplane status.

**Why custom health checks?** These resources represent your application infrastructure. Detailed health checks provide:
- Clear visibility into provisioning status
- Propagation of child resource health to parent
- Early warning of configuration errors

### Crossplane Infrastructure Components

#### Providers (pkg.crossplane.io/Provider)
- **Uses ArgoCD built-in health check** from [upstream](https://github.com/argoproj/argo-cd/blob/master/resource_customizations/pkg.crossplane.io/Provider/health.lua)
- **Healthy**: When `Installed=True` AND `Healthy=True`
- **Degraded**: If either condition is False
- **Progressing**: Waiting for installation

**No custom configuration needed** - ArgoCD has native support.

#### Functions (pkg.crossplane.io/Function)
- **Custom health check** (minimal implementation)
- **Healthy**: When `Installed=True` AND `Healthy=True`
- **Progressing**: Installation in progress

**Why custom?** Functions don't have ArgoCD built-in support yet.

#### CompositeResourceDefinitions (XRDs)
- **No health checks configured** - XRDs are schema definitions
- ArgoCD treats them as standard Kubernetes CRDs
- Once applied, they're established in the API server

**Why no health check?** XRDs are infrastructure metadata, not runtime resources. Once synced, they're operational.

#### Compositions
- **No health checks configured** - Compositions are templates
- Similar to XRDs, they're configuration not runtime state

#### Managed Resources (*.upbound.io/*)
- **Custom health check** for provider-specific managed resources
- **Healthy**: When `Synced=True` AND (`Ready=True` OR `Healthy=True`)
- **Degraded**: If `Synced=False`
- **Progressing**: Waiting for conditions

### Status Propagation

Crossplane's `function-auto-ready` propagates child resource status to parent composites:

```
AppEnvironment (Tracked in Git by ArgoCD)
├── GCPProject (Dynamic child) → Ready=True
├── GCPArtifact (Dynamic child) → Ready=True
├── GCPSecretManager (Dynamic child) → Ready=True
└── GCPCloudRun (Dynamic child) → Ready=True
    ↓
AppEnvironment: Ready=True (aggregated status)
```

**Key insight:** ArgoCD only tracks parent XRs in Git. Child resources are dynamically created by Crossplane. Health status bubbles up through composition functions, giving you a single health indicator for the entire infrastructure stack.

## References

- [Official Crossplane + ArgoCD Guide](https://docs.crossplane.io/latest/guides/crossplane-with-argo-cd/)
- [ArgoCD Resource Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
- [Crossplane Status Propagation](https://docs.crossplane.io/latest/concepts/composite-resources/#status)
