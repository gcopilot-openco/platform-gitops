# App Environments

Service instances provisioned through the platform.

## Naming

`{service}-{environment}.yaml` (e.g., `payment-service-hml.yaml`)

## Create New Environment

1. Copy example or create new file
2. Configure:
   ```yaml
   apiVersion: platform.openco.tech/v1alpha1
   kind: AppEnvironment
   metadata:
     name: my-service-hml
     namespace: apps
   spec:
     appName: my-service
     environment: hml        # sandbox, hml, prod
     teamId: my-team
     costCenter: CC-1234
     region: us-east4
     cloudRun:               # optional
       enabled: true
       port: 8080
   ```
3. Commit to main -> ArgoCD syncs

## Verify

```bash
kubectl get appenvironment my-service-hml -w
kubectl describe appenvironment my-service-hml
```

## Provisioned Resources

AppEnvironment orchestrates:
- GCPProject (prj-{env}-{app})
- GCPIdentity (service accounts)
- GCPArtifact (container registry)
- GCPCloudRun (service)
- GCPSecretManager (secrets)
