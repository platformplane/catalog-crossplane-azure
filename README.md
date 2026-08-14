# Azure Crossplane Self-Service Catalog Items

Azure-specific items for the Crossplane Service Catalog. See [`catalog-crossplane`](https://github.com/platformplane/catalog-crossplane) for general documentation.

## Azure Function (Flex Consumption)

`AzureFunctionFlexConsumption` creates a Linux Function App on a Flex Consumption (`FC1`) plan. Custom container images are not supported.

Configure these fields:

- `location`: Choose during creation; it cannot be changed.
- `runtimeName`: Select the runtime.
- `runtimeVersion`: Override its version when needed.
- `environmentVariables`: Add non-secret values only.
- `package`: Provide the full HTTPS URL of a ready-to-run ZIP in the current space's private GitLab Generic Package registry.
- `packageSha256`: Provide a 64-character lowercase SHA-256 digest when integrity verification is required.

The deployment validates the ZIP and its digest when provided, then deploys it using the cluster's managed identity. Use a new package URL and update its digest if set to deploy a new version.

Each function gets a private, dedicated storage account. Its system-assigned identity receives Blob Data Owner access on that account for `AzureWebJobsStorage`; Queue and Table access are not granted.

The connection Secret contains the `default` host key as `function-key`, refetched whenever a new package is deployed. Master and system keys are excluded.

Optional access configuration:

- `managedIdentityRoleAssignments`: Assign an approved role to an existing resource and scope it to that resource. The current approved role manages AKS.

See [`examples/azurefunctionflexconsumption.yaml`](examples/azurefunctionflexconsumption.yaml).

## Related Infrastructure

The generic `Network` resource is managed by `crossplane-azure-operator`, not this catalog. Its current subnet declarations support App Service and Flex Consumption workloads.

## Azure Storage Accounts

`AzureStorageV2` creates private endpoints for Blob, File, Queue, and Table storage. Keep `public` disabled to prevent public network access.
