# Crossplane Status Propagation: The "Viktor Pattern"

Based on the patterns popularized by Viktor Farcic (DevOps Toolkit) and used in advanced CNOE-style platforms, status propagation is the "secret sauce" for a high-quality Internal Developer Platform (IDP).

## Core Principles

### 1. The XR as a "Glass Pane"
The high-level Composite Resource (XR) should not just be a request form; it should be a "Single Glass Pane" for the developer. They should never need to run `kubectl get managed` or `kubectl describe` on low-level resources.

### 2. Granular Status Fields
Instead of a single `Ready` boolean, the XRD should define status fields for each functional component.
- `projectStatus`
- `registryStatus`
- `identityStatus`
- `computeStatus`

### 3. Error Mirroring
When a low-level resource fails, the error message should be patched up to the XR status. This is done by patching from `status.conditions[?(@.type=='Ready')].message` to a status field in the XR.

### 4. Explicit Readiness Checks
Relying on `function-auto-ready` is often not enough for complex compositions. Explicit `readinessChecks` allow the parent XR to stay in a `Progressing` state until *exactly* the right conditions are met in the children.

### 5. CLI UX: Additional Printer Columns
The XRD should define `additionalPrinterColumns` so that `kubectl get` provides immediate, high-value feedback.

---

## Proposed Architecture for AppEnvironment

### XRD Updates
- Add `projectStatus`, `identityStatus`, `artifactStatus`, `cloudRunStatus`.
- Add printer columns for `READY`, `SYNCED`, `PROJECT`, and `CLOUD-RUN-URL`.

### Composition Updates
- Add `readinessChecks` for all child XRs.
- Add `ToCompositeFieldPath` patches for:
    - `status.conditions[?(@.type=='Ready')].status` -> component status
    - `status.conditions[?(@.type=='Ready')].message` -> component status (transforming to message if failed)
- Ensure all important URLs and IDs are propagated.

## Why this matters for the Platform
- **Self-Service Support**: Developers can debug their own infrastructure failures.
- **Observability**: Platform teams can see at a glance why environments are failing without deep diving into GCP.
- **Automation**: CI/CD pipelines can wait for specific status fields before proceeding to the next step.
