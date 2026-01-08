# Tenant Environment Resources

This directory contains TenantEnvironment composite resources (XRs) for all services provisioned through the platform.

## Naming Convention

`{service-name}-{environment}.yaml`

### Examples
- `payment-service-hml.yaml` - Payment service in homologation
- `auth-service-prod.yaml` - Auth service in production

## Structure

Each TenantEnvironment XR automatically provisions:

| Resource | Name | Purpose |
|----------|------|---------|
| **GCP Project** | `prj-{env}-{service-name}` | Isolated GCP project for the service |
| **Service Account** | `sa-{service-name}` | Runtime identity for authentication |
| **Artifact Registry** | `repo-{service-name}` | Docker image repository (us-central1, Docker format) |
| **APIs** | compute, run, artifactregistry, iam | Required GCP services automatically enabled |

## How to Add a New Environment

### Option 1: Via Backstage (Recommended - Future)
1. Use the "Microservice (Crossplane)" template in Backstage
2. Fill the form with service details
3. Backstage generates `k8s/resources.yaml`
4. Copy to `claims/{service-name}-{env}.yaml`
5. Create PR to platform-gitops

### Option 2: Manual
1. Copy `examples/demo-service-hml.yaml`
2. Update the following fields:
    ```yaml
    metadata.name: {service-name}-{environment}
    spec.appName: {service-name}
    spec.environment: {environment}
    spec.teamId: {team-id}
    spec.costCenter: {cost-center}
    ```
3. Create PR to platform-gitops
4. Merge after approval

## Verification

After merging PR, verify provisioning:

```bash
# Get XR status (watch updates)
kubectl get tenantenvironment {service-name}-{env} -w

# Detailed status
kubectl describe tenantenvironment {service-name}-{env}

# Check GCP resources
gcloud projects list --filter="name:prj-{env}-{service-name}"
gcloud iam service-accounts list --filter="email:sa-{service-name}@"
gcloud artifacts repositories list --location=us-central1 --filter="repositoryId:repo-{service-name}"
```

## Troubleshooting

### XR Stuck in Pending

```bash
kubectl describe tenantenvironment {service-name}-{env}
# Check "Events" section for error messages
```

### GCP Resources Not Created

1. Verify XR exists: `kubectl get tenantenvironment`
2. Check Kyverno policies: `kubectl get clusterpolicy`
3. Check Crossplane logs: `kubectl logs -n crossplane-system deployment/crossplane`
4. Verify GCP ProviderConfig: `kubectl get providerconfig -n crossplane-system`

## Best Practices

- Always specify all required fields in the XR
- Use meaningful cost-center values for billing tracking
- Ensure team name matches your organization's team structure
- Test in `hml` (homologation) before deploying to `prod`
- XRs are namespaced - apply to the appropriate namespace

## Support

- **Issues**: Create an issue in the platform-gitops repository
- **Questions**: Contact the platform team in #platform-support
