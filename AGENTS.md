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

## Status Propagation Pattern

All compositions implement a "Developer" condition to surface user-friendly errors:

```yaml
# In each composition's SetConditions:
- condition: Developer
  target: CompositeAndClaim
  type: MatchConditionFromFieldPath
  fieldPath: status.conditions[?(@.type=='Synced')]
  fallbackTo: Status
```

To extract actual GCP API error messages, use regex:
```yaml
message:
  type: MatchConditionFromFieldPath
  fieldPath: status.conditions[?(@.type=='Synced')]
  fallbackTo: Status
  regexp: "(?P<ErrorMsg>.*)"
```

## Non-Obvious Crossplane Gotchas

### String Transforms Require Type Field
String transforms with `fmt` MUST include `type: Format`:
```yaml
# WRONG - silently fails
- type: string
  string:
    fmt: "prj-%s-%s"

# CORRECT
- type: string
  string:
    type: Format
    fmt: "prj-%s-%s"
```

### Deletion Policy for GCP Projects
Use `deletionPolicy: Abandon` on GCP Projects to avoid 30-day deletion hold:
```yaml
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.deletionPolicy
    toFieldPath: spec.deletionPolicy
    transforms:
      - type: map
        map:
          Delete: Abandon  # Map to Abandon to avoid 30-day hold
          Orphan: Orphan
```

### XRD additionalPrinterColumns
Always add READY and SYNCED columns to XRDs for better `kubectl get` output:
```yaml
additionalPrinterColumns:
  - name: SYNCED
    type: string
    jsonPath: .status.conditions[?(@.type=='Synced')].status
  - name: READY
    type: string
    jsonPath: .status.conditions[?(@.type=='Ready')].status
  - name: AGE
    type: date
    jsonPath: .metadata.creationTimestamp
```

### ClusterProviderConfig Location
`ClusterProviderConfig` resources must be in `crossplane/providers/` directory (not a separate `providerconfigs/` dir) to be picked up by the providers ArgoCD app.

### ArgoCD Health Checks for Crossplane
Use the official Crossplane health check Lua scripts. Key insight: some resources like `ClusterProviderConfig` have `status.users` instead of `status.conditions` - the health check must handle both cases.

## Resource Sequencing in AppEnvironment

The AppEnvironment composition creates resources in this order:
1. **GCPProject** - Creates the GCP project and enables APIs
2. **GCPIdentity** - Creates service accounts (depends on project)
3. **GCPArtifact** - Creates artifact registry (depends on project)
4. **GCPCloudRun** - Creates Cloud Run services (depends on identity + artifact)
5. **GCPSecretManager** - Creates secret manager (depends on project)
