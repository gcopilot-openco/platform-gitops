# AGENTS.md - Agentic Coding Guidelines

This document provides guidance for AI coding agents working in this repository.

## Repository Overview

This is a **GitOps infrastructure repository** for managing GCP resources using the BACK stack:
- **B**ackstage, **A**rgoCD, **C**rossplane, **K**ubernetes

**Important**: This repository contains only YAML configuration files. There is no application code, build system, or test suite.

## Build/Lint/Test Commands

### No Traditional Build System

This repository has no `package.json`, `Makefile`, or similar. All validation happens through:
1. **ArgoCD Sync** - Automatic deployment on merge to `main`
2. **Kubernetes API** - Schema validation when resources are applied
3. **Kyverno Policies** - Runtime policy enforcement

### Local Validation Commands

```bash
# Validate YAML syntax
yq eval '.' <file>.yaml

# Render Crossplane composition locally (dry-run)
crossplane beta render <claim>.yaml <composition>.yaml <functions>.yaml

# Check ArgoCD application status
kubectl get applications -n argocd

# View Crossplane XR status
kubectl get appenvironments -A
kubectl get gcpprojects -A
```

### Running a Single Test

There are no automated tests. To validate a specific resource:

```bash
# Validate a single YAML file's syntax
yq eval '.' crossplane/definitions/app-environment-xrd.yaml

# Check if a specific Crossplane XR is ready
kubectl get appenvironment <name> -n <namespace> -o yaml

# Check ArgoCD sync status for a specific app
argocd app get <app-name>
```

## Code Style Guidelines

### YAML Formatting

| Rule | Standard |
|------|----------|
| Indentation | 2 spaces (no tabs) |
| Line length | No strict limit, prefer readability |
| Trailing newline | Yes, single newline at end of file |
| Document separator | Use `---` between multiple documents |

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| XRD names | lowercase with dots | `appenvironments.platform.openco.tech` |
| Composition names | lowercase with dots | `gcpproject.platform.openco.tech` |
| Resource names | kebab-case | `gcp-project-xr`, `compute-api` |
| Spec field names | camelCase | `appName`, `teamId`, `costCenter` |
| Labels | kebab-case | `team`, `env`, `cost-center` |
| File names | kebab-case | `app-environment-xrd.yaml` |
| API groups | lowercase with dots | `platform.openco.tech` |

### String Quoting

```yaml
# Prefer unquoted strings when possible
environment: prod
region: us-east4

# Use double quotes for special values
status: "True"
port: "8080"  # when string type required
```

### Comment Style

```yaml
# Section headers use numbered format
# 1. Create GCPProject XR
- name: gcp-project-xr
  base:
    apiVersion: platform.openco.tech/v1alpha1
    kind: GCPProject

# Inline comments for clarification
autoCreateNetwork: false  # Managed via SharedVPC
```

## Repository Structure

```
platform-gitops/
├── argocd/
│   ├── apps/
│   │   ├── app-of-apps.yaml          # Root Application
│   │   └── applications/             # Child Applications
│   └── config/                       # ArgoCD configuration
├── crossplane/
│   ├── providers/                    # Provider packages & configs
│   ├── definitions/                  # CompositeResourceDefinitions (XRDs)
│   ├── compositions/                 # Composition templates
│   ├── composites/                   # App environment instances
│   └── providerconfigs/              # Provider credentials
├── kyverno/
│   └── policies/                     # Policy definitions
└── docs/                             # Documentation
```

## Crossplane Patterns

### XRD Structure

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: <plural>.<group>
spec:
  group: platform.openco.tech
  names:
    kind: <Kind>
    plural: <plural>
  scope: Namespaced  # Always namespaced for app resources
  defaultCompositionRef:
    name: <kind-lowercase>.<group>
```

### Composition Structure

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: <kind-lowercase>.platform.openco.tech
spec:
  compositeTypeRef:
    apiVersion: platform.openco.tech/v1alpha1
    kind: <Kind>
  mode: Pipeline  # Always use Pipeline mode
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          # ... resources here
    - step: auto-ready  # Always last step
      functionRef:
        name: function-auto-ready
```

### Readiness Checks

```yaml
readinessChecks:
  - type: MatchCondition
    matchCondition:
      type: Ready
      status: "True"
```

## Domain Concepts

### Core XRD Types (platform.openco.tech/v1alpha1)

| Kind | Purpose |
|------|---------|
| `AppEnvironment` | Top-level orchestrator for service environments |
| `GCPProject` | GCP Project + standard APIs |
| `GCPIdentity` | Service Accounts + IAM bindings |
| `GCPArtifact` | Artifact Registry repositories |
| `GCPCloudRun` | Cloud Run services |
| `GCPSecretManager` | Secret Manager entries |

### Environment Values

- `sandbox` - Experimental/development
- `hml` - Homologation/staging
- `prod` - Production

### Required Fields for AppEnvironment

```yaml
spec:
  appName: payment-service     # kebab-case, 1-63 chars
  environment: hml             # sandbox | hml | prod
  teamId: finance              # Team identifier
  costCenter: CC-1234          # Cost center code
```

### Naming Patterns

| Resource | Pattern | Example |
|----------|---------|---------|
| GCP Project | `prj-{env}-{appName}` | `prj-hml-payment-service` |
| Artifact Repo | `repo-{appName}` | `repo-payment-service` |
| Cloud Run | `{appName}` | `payment-service` |

## Critical Rules for Agents

1. **Never use `kubectl apply` directly** - All changes go through Git commits
2. **Test in sandbox/hml first** - Never deploy directly to prod
3. **ArgoCD auto-syncs from main** - Merging to main triggers deployment
4. **Sync waves matter** - Order: providers (1) -> definitions (2) -> compositions (3) -> composites (4)
5. **Validate YAML before committing** - Use `yq` to check syntax
6. **Keep resources idempotent** - Compositions should be safe to re-apply
7. **Use status propagation** - Always propagate important status fields to composite
