# ArgoCD Configuration

Configuration patches for Crossplane integration.

## argocd-cm-patch.yaml

Configures:
- Resource tracking: Annotation-based (required for Crossplane)
- Resource exclusions: Excludes ProviderConfigUsage from UI
- Health checks: Custom Lua scripts for platform.openco.tech resources

## Apply Configuration

```bash
kubectl patch configmap argocd-cm -n argocd --type=merge --patch-file=argocd/config/argocd-cm-patch.yaml
kubectl rollout restart statefulset argocd-application-controller -n argocd
kubectl rollout restart deployment argocd-server -n argocd
```

## Health Check Logic

Custom health checks for composite resources (platform.openco.tech/*):
- **Healthy**: `Synced=True` AND `Ready=True`
- **Progressing**: Waiting for conditions
- **Degraded**: Either condition is False
