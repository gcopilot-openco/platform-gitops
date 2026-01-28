# Platform GitOps

Single source of truth for Open-Co platform infrastructure, managed by ArgoCD.

## Stack

**BACK Stack**: Backstage, ArgoCD, Crossplane, Kubernetes

| Component | Version | Purpose |
|-----------|---------|---------|
| ArgoCD | v7.7.10 | GitOps continuous deployment |
| Crossplane | v2.1.1 | Infrastructure as Code for GCP |
| Kyverno | v3.3.0 | Policy enforcement |
| Backstage | Planned | Developer portal |

## Current Status

**Phase 2 Complete** - Service composition with nested XRD architecture operational.

See [docs/ROADMAP.md](docs/ROADMAP.md) for project phases and future plans.

## Structure

```
platform-gitops/
├── argocd/apps/           # ArgoCD applications (app-of-apps)
├── crossplane/
│   ├── providers/         # GCP providers & functions
│   ├── definitions/       # XRDs
│   ├── compositions/      # Composition templates
│   └── composites/        # Service instances
├── kyverno/policies/      # Governance policies
└── docs/                  # Architecture & roadmap
```

## Deploy New Service

1. Create: `crossplane/composites/{service}-{env}.yaml`
2. Configure:
   ```yaml
   apiVersion: platform.openco.tech/v1alpha1
   kind: AppEnvironment
   metadata:
     name: my-service-hml
     namespace: apps
   spec:
     appName: my-service
     environment: hml          # sandbox, hml, prod
     teamId: my-team
     costCenter: CC-1234
     region: us-east4
   ```
3. Commit to `main` -> ArgoCD syncs automatically
4. Verify: `kubectl get appenvironment my-service-hml -w`

## Monitor

```bash
kubectl get applications -n argocd          # ArgoCD apps
kubectl get appenvironment -A               # All environments
kubectl describe appenvironment <name>      # Details
```

## Rules

- **Never `kubectl apply`** - Use Git commits
- **Test first** - Deploy to sandbox/hml before prod
- **ArgoCD auto-syncs** - Merge to main triggers deployment

## Links

- ArgoCD UI: https://argocd.alm.open-co.tech
- Cluster: openco_evergreen_cluster (us-east-1)
