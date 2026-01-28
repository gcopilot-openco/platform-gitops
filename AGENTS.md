# AGENTS.md - Agentic Coding Guidelines

GitOps infrastructure repository for GCP resources using the BACK stack (Backstage, ArgoCD, Crossplane, Kubernetes).

**This repository contains only YAML configuration files. No application code, build system, or test suite.**

## Quick Reference

```bash
# Validate YAML syntax
yq eval '.' <file>.yaml

# Render Crossplane composition (dry-run)
crossplane beta render <claim>.yaml <composition>.yaml <functions>.yaml

# Check ArgoCD sync status
kubectl get applications -n argocd

# Check Crossplane XR status
kubectl get appenvironment <name> -n <namespace> -o yaml
```

## Critical Rules

1. **Never use `kubectl apply` directly** - All changes go through Git commits
2. **Test in sandbox/hml first** - Never deploy directly to prod
3. **ArgoCD auto-syncs from main** - Merging to main triggers deployment
4. **Sync waves matter** - Order: providers (1) -> definitions (2) -> compositions (3) -> composites (4)
5. **Validate YAML before committing** - Use `yq` to check syntax

## Repository Structure

```
platform-gitops/
├── argocd/
│   ├── apps/                    # ArgoCD applications (app-of-apps pattern)
│   └── config/                  # ArgoCD configuration
├── crossplane/
│   ├── providers/               # GCP providers & functions
│   ├── definitions/             # XRDs (CompositeResourceDefinitions)
│   ├── compositions/            # Composition templates
│   └── composites/              # App environment instances
├── kyverno/policies/            # Policy definitions
└── docs/                        # Architecture & roadmap
```

## YAML Style

| Rule | Standard |
|------|----------|
| Indentation | 2 spaces (no tabs) |
| Trailing newline | Yes |
| Document separator | `---` between multiple documents |

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| XRD names | lowercase.dots | `appenvironments.platform.openco.tech` |
| Resource names | kebab-case | `gcp-project-xr` |
| Spec fields | camelCase | `appName`, `teamId` |
| File names | kebab-case | `app-environment-xrd.yaml` |

## Required Labels

```yaml
labels:
  team: <team-id>
  env: <environment>        # sandbox, hml, prod
  cost-center: <cc-code>
```

## Crossplane Patterns

All compositions use function pipeline:
```yaml
pipeline:
  - step: patch-and-transform    # Core resource creation
  - step: auto-ready             # Always last
```

Common patches:
```yaml
# Copy from composite to composed
- fromFieldPath: spec.appName
  toFieldPath: spec.appName

# Copy from composed to composite status
- type: ToCompositeFieldPath
  fromFieldPath: status.projectId
  toFieldPath: status.projectId
```

Readiness checks (always required):
```yaml
readinessChecks:
  - type: MatchCondition
    matchCondition:
      type: Ready
      status: "True"
```

## ArgoCD Sync Waves

| Wave | Resources |
|------|-----------|
| 1 | Providers & Functions |
| 2 | XRDs (Definitions) |
| 3 | Compositions |
| 4 | XRs (Instances) |

## Core XRD Types

| Kind | Purpose |
|------|---------|
| `AppEnvironment` | Top-level orchestrator |
| `GCPProject` | GCP Project + APIs |
| `GCPIdentity` | Service Accounts + IAM |
| `GCPArtifact` | Artifact Registry |
| `GCPCloudRun` | Cloud Run services |
| `GCPSecretManager` | Secret Manager |

## Environments

- `sandbox` - Development/experimentation
- `hml` - Homologation/staging
- `prod` - Production

## Naming Patterns

| Resource | Pattern |
|----------|---------|
| GCP Project | `prj-{env}-{appName}` |
| Artifact Repo | `repo-{appName}` |
| Cloud Run | `{appName}` |
