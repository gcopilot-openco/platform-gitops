# Platform GitOps - Infrastructure Source of Truth

This repository is the single source of truth for all Open-Co platform infrastructure, managed by ArgoCD.

## 🎯 Purpose

Manage BACK stack (Backstage, ArgoCD, Crossplane, Kubernetes) infrastructure definitions using GitOps principles. All changes are version-controlled, reviewed, and automatically deployed to the cluster.

## 📁 Directory Structure

### ArgoCD Configuration
- `argocd/apps/` - ArgoCD applications and sync configuration
- `argocd/config/` - ArgoCD system configuration

### Crossplane Infrastructure
- `crossplane/providers/` - GCP provider packages and configuration
- `crossplane/definitions/` - CompositeResourceDefinitions (XRDs)
- `crossplane/compositions/` - Composition templates that bundle resources
- `crossplane/composites/` - App environment manifests (instances of services)

### Policy Enforcement
- `kyverno/policies/` - Kubernetes admission policies for governance

### Documentation
- `docs/` - Platform architecture and deployment guides
  - `ARCHITECTURE.md` - System design and data flow
  - `PHASE1_COMPLETE.md` - Phase 1 summary
  - `PHASE2_DEPLOYMENT.md` - Phase 2 deployment verification
  - `DEVELOPER_GUIDE.md` - How to provision infrastructure

## 🚀 Quick Start

### Deploy New Service Infrastructure

1. Create app file: `crossplane/composites/{service-name}-{environment}.yaml`
2. Base on: `crossplane/composites/examples/full-app-production.yaml`
3. Update fields:
   - `metadata.name`
   - `spec.appName`
   - `spec.environment` (hml, prod, or sandbox)
   - `spec.teamId`
   - `spec.costCenter`
   - `spec.cloudRun` (optional)
   - `spec.secretManager` (optional)
4. Create PR → Merge → ArgoCD syncs automatically
5. Verify: `kubectl get appcomposite {service-name}-{env}`

### Monitor Deployments

```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Watch app provisioning
kubectl get appcomposite -w

# View cluster events
kubectl get events -A --sort-by='.lastTimestamp'
```

## 🔄 How It Works

1. **You**: Commit to platform-gitops repo
2. **GitHub**: Webhook notifies ArgoCD
3. **ArgoCD**: Detects changes and syncs cluster
4. **Kyverno**: Validates policies
5. **Crossplane**: Reconciles resources
6. **GCP**: Resources provisioned

## 📊 Current Status

### ✅ Phase 1: Infrastructure (Complete)
- ArgoCD v7.7.10
- Crossplane v2.1.1
- GCP Providers v2.4.0
- Kyverno v3.3.0
- Basic policies

### ✅ Phase 2: Service Composition (Complete)
- Nested AppEnvironment XRD architecture
- Modular Compositions (Project, Identity, Artifact, Cloud Run, Secrets)
- Enhanced Kyverno policies
- Automated readiness via function-auto-ready

### 🔮 Phase 3: Backstage Integration (Planned)
- Template-based service creation
- Automated manifest generation
- Status feedback loop

## 📚 Documentation

- **Getting Started**: See `docs/DEVELOPER_GUIDE.md`
- **Architecture**: See `docs/ARCHITECTURE.md`
- **Phase 2 Verification**: See `docs/PHASE2_DEPLOYMENT.md`
- **Provisioning Apps**: See `crossplane/composites/README.md`

## 🛠️ Maintenance

### Check Health

```bash
source /Users/filipe/Desktop/open/platform/open-platform/.sisyphus/plans/.eks-cluster-creds

# Check all applications
kubectl get applications -n argocd

# Check resource status
kubectl get all -A

# Check policies
kubectl get clusterpolicy
```

### Update Configuration

1. Edit files locally
2. Commit to main branch: `git push origin main`
3. ArgoCD syncs automatically (~1 minute)
4. Monitor: `kubectl get applications -n argocd -w`

### Rollback Changes

```bash
git revert HEAD
git push origin main
# ArgoCD will auto-sync to previous state
```

## ⚠️ Important Notes

- **Never use `kubectl apply` directly** - Use git commits instead
- **Always test in `hml` or `sandbox` environment first** - Before pushing to `prod`
- **Review ArgoCD status** - Before and after changes
- **Check Kyverno policies** - They may modify or reject resources

## 📞 Support

- **Documentation**: Read `docs/` directory
- **Issues**: Create GitHub issue in this repository
- **Questions**: Contact platform team in #platform-support Slack channel
- **ArgoCD UI**: https://argocd.alm.open-co.tech

---

**Last Updated**: 2026-01-08  
**Cluster**: openco_evergreen_cluster (us-east-1)  
**Status**: Phase 2 active
