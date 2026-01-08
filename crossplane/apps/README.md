# App Environment Resources

This directory contains `AppClaim` resources for all services provisioned through the platform.

## Naming Convention

`{service-name}-{environment}.yaml`

### Examples
- `payment-service-hml.yaml` - Payment service in homologation
- `auth-service-prod.yaml` - Auth service in production
- `data-pipeline-sandbox.yaml` - Experimental environment

## Structure

The platform uses a **Nested Composite Resource** architecture. A single `AppEnvironment` orchestrates multiple domain-specific XRs:

| Resource | Scope | Purpose |
|----------|-------|---------|
| **GCPProject** | Internal | Provisions GCP Project + enabled APIs (Compute, Run, IAM, etc.) |
| **GCPIdentity** | Internal | Provisions Service Accounts and IAM role bindings |
| **GCPArtifact** | Internal | Provisions Artifact Registry repositories (Default: us-east4) |
| **GCPSecretManager** | Internal | Provisions Secret Manager entries for sensitive configuration |
| **GCPCloudRun** | Internal | Provisions Cloud Run v2 services and invoker permissions |

## How to Add a New Environment

### Option 1: Via Backstage (Recommended)
1. Use the "Microservice (Crossplane)" template in Backstage.
2. Fill the form with service details.
3. Backstage generates the app manifest.
4. Create PR to `platform-gitops` adding the file to `crossplane/apps/`.

### Option 2: Manual
1. Copy `examples/demo-service-sandbox.yaml`.
2. Update the following fields:
    ```yaml
    metadata.name: {service-name}-{environment}
    spec.appName: {service-name}
    spec.environment: hml | prod | sandbox
    spec.teamId: {team-id}
    spec.costCenter: {cost-center}
    spec.region: us-east4 # optional
    spec.cloudRun: # optional
      enabled: true
      port: 8080
    ```
3. Create PR to `platform-gitops`.
4. Merge after approval.

## Verification

After merging PR, verify provisioning:

```bash
# Get Claim status
kubectl get appclaim {service-name}-{env} -w

# Check orchestrated XRs
kubectl get gcpproject,gcpidentity,gcpartifact,gcpcloudrun -l crossplane.io/composite={composite-id}

# Check GCP resources
gcloud projects list --filter="name:prj-{env}-{service-name}"
gcloud artifacts repositories list --location=us-east4 --filter="repositoryId:repo-{service-name}"
```

## Troubleshooting

### Claim Stuck in Progress
Check if the top-level Composite (XR) is healthy:
```bash
kubectl describe appclaim {service-name}-{env}
```

Check the child XRs for specific domain errors:
```bash
kubectl get gcpproject -l crossplane.io/claim-name={service-name}-{env}
```

## Best Practices

- **Test First**: Always deploy to `sandbox` or `hml` before `prod`.
- **Labels**: Ensure `teamId` and `costCenter` are accurate for billing attribution.
- **Regions**: Default is `us-east4` to ensure low latency and regional compliance.
- **Security**: Use the `secretManager` block to prepare slots for sensitive data.

## Support

- **Issues**: Create an issue in the platform-gitops repository.
- **Questions**: Contact the platform team in #platform-support.
