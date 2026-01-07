# Platform GitOps

BACK Stack configuration for Open Co Internal Developer Platform:
- **B**ackstage - Developer portal
- **A**rgoCD - GitOps continuous delivery
- **C**rossplane - Infrastructure provisioning
- **K**yverno - Policy enforcement

## Structure

- `argocd/` - ArgoCD applications and configuration
- `crossplane/` - Crossplane providers and compositions
- `kyverno/` - Kyverno policies
- `claims/` - Crossplane claims for service infrastructure

## Setup

All components are managed by ArgoCD using the App-of-Apps pattern.
