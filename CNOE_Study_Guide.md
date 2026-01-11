# CNOE Comprehensive Study Guide
## Cloud Native Operational Excellence

### 1. Executive Summary
**CNOE (pronounced "Kuh-noo")** is an open-source framework and community collaboration designed to help organizations build Internal Developer Platforms (IDPs) using proven CNCF technologies.

*   **Goal**: To de-risk technology bets by creating a consensus-driven "Golden Stack" for large-scale enterprises.
*   **Core Philosophy**: Open-source first, community-driven, and focused on tools rather than prescribing internal business practices.
*   **The "BACK" Stack**: CNOE is fundamentally built on **B**ackstage, **A**rgoCD, **C**rossplane (or Terraform), and **K**eycloak/Kubernetes.

---

### 2. Architecture & Capabilities
CNOE categorizes the IDP into several "Capability" verticals to ensure a modular yet cohesive system.

| Capability | Recommended Technology | Role in CNOE |
| :--- | :--- | :--- |
| **Developer Portal** | **Backstage** | The entry point for developers; handles service catalog and scaffolding. |
| **Continuous Delivery** | **ArgoCD / Flux** | The GitOps engine that reconciles state from Git to the cluster. |
| **Infrastructure as Code** | **Crossplane / Terraform** | Manages cloud resources (S3, RDS, Projects) via Kubernetes APIs. |
| **Identity & Access** | **Keycloak** | Centralized OIDC provider for SSO across the entire platform. |
| **Secret Management** | **External Secrets (ESO)** | Syncs secrets from external vaults (AWS SM, Vault) into K8s. |
| **Policy & Governance** | **Kyverno** | Enforces "Guardrails" (e.g., mandatory tags, approved registries). |
| **Workflow Orchestration** | **Argo Workflows** | Handles complex, imperative tasks (CI pipelines, data processing). |

#### The "GitOps Bridge" Pattern
CNOE uses the **App-of-Apps** or **ApplicationSet** pattern to manage the platform itself. The platform is treated as a set of versioned artifacts in a Git repository (`platform-gitops`), allowing the IDP to be upgraded and managed using the same GitOps workflows it provides to developers.

---

### 3. The IDPBuilder Tool
**idpBuilder** is a CLI tool designed to provide an "IDP in a box" for local development and testing.

*   **Components**: It spins up a **Kind** cluster, installs **Gitea** (local Git + OCI registry), **ArgoCD**, and **Nginx Ingress**.
*   **Key Feature**: It supports a `cnoe://` protocol for ArgoCD, allowing you to sync local directories directly into the cluster's internal Git server for rapid iteration.
*   **Routing**: Supports both domain-based (`argocd.cnoe.localtest.me`) and path-based routing (crucial for GitHub Codespaces).

---

### 4. Reference Implementations

#### A. Local Implementation
Designed for "Day 0" exploration. It provides a pre-configured Backstage instance with "Golden Path" templates that demonstrate:
1.  Scaffolding a new service.
2.  Provisioning a local Git repo.
3.  Registering the app in ArgoCD.

#### B. AWS Reference Implementation
A production-grade architecture for EKS.
*   **IAM**: Uses **EKS Pod Identity** (or IRSA) to grant fine-grained permissions to Crossplane and ESO.
*   **DNS**: Uses **ExternalDNS** with Route53.
*   **TLS**: Uses **Cert-Manager** with Let's Encrypt.
*   **Bootstrapping**: Uses a `scripts/install.sh` to bootstrap ArgoCD and ESO, which then take over the management of all other platform components.

#### C. Azure Reference Implementation
A similar production-grade architecture for AKS.
*   **Workload Identity**: Uses Azure Workload Identity for secure cloud access.
*   **Tooling**: Relies on **Taskfile** and **Helmfile** for the orchestration of the platform installation.
*   **Secret Management**: Integrates deeply with **Azure Key Vault**.

---

### 5. Extensibility: Stacks & Plugins

#### CNOE Stacks
A "Stack" is a collection of ArgoCD applications that add specific capabilities to the IDP.
*   **Terraform Integration**: Adds the `tf-controller` to run TF modules via GitOps.
*   **Kyverno Stack**: Adds centralized policy enforcement.
*   **Vcluster Stack**: Enables multi-tenancy by spinning up virtual clusters for different environments.

#### CNOE CLI & Plugins
*   **Template Generation**: The CNOE CLI can scan Kubernetes CRDs or Terraform modules and automatically generate **Backstage Scaffolder Templates**.
*   **Custom Plugins**: CNOE provides Backstage plugins for **Terraform status**, **Argo Workflows visibility**, and **Apache Spark** monitoring.

---

### 6. Operational Guides & Best Practices

#### Security Principles
1.  **Identity Federation**: Always federate Keycloak with your corporate OIDC/AD.
2.  **No Secrets in Git**: Use External Secrets Operator. Even for local dev, `idpbuilder` manages secrets via Kubernetes secrets that never leave the ephemeral environment.
3.  **Governance as Code**: Use Kyverno to prevent "shadow IT" by blocking resources that don't comply with corporate standards at the API admission level.

#### Troubleshooting Workflow
1.  **Check the "Source of Truth"**: Verify if the Git repository has the expected manifests.
2.  **Verify ArgoCD Sync**: Check the `Application` status. Look for `OutOfSync` or `Degraded`.
3.  **Check Secret Propagation**: Verify the `ExternalSecret` resource status.
4.  **Check DNS/Certs**: For UI issues, check `Certificate` and `Challenge` resources in the `cert-manager` namespace.

---
**Document Status**: Final Study Guide (Version 1.0)
**Source**: [cnoe.io](https://cnoe.io)
