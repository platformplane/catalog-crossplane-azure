# Azure Crossplane Self-Service Catalog Items

This repo contains the Crossplane Service Catalog items that are specific to Azure.

For more information, have a look at the [catalog-crossplane](https://github.com/platformplane/catalog-crossplane) repository.

## Azure Function (Flex Consumption)

`AzureFunctionFlexConsumption` provisions a Linux Function App on a Flex Consumption (`FC1`) plan: fast, pay-per-use elastic scaling, with no billed instance kept warm while idle. Set `runtimeName`/`runtimeVersion` (e.g. `node`/`20` or `custom`/`1.0`) and add any non-secret environment variables. Custom container images are not supported.

Choose the Azure `location` during creation. It cannot be changed afterward; the catalog portal displays it as read-only during updates and Kubernetes rejects changes made through other clients.

Each function gets its own dedicated, private storage account for Function host metadata and its deployment package container. The composition configures identity-based `AzureWebJobsStorage` and grants the Function App's system-assigned identity Blob Data Owner on that dedicated account via a Crossplane-managed `RoleAssignment`. Queue and Table access is not provisioned because the HTTP and timer triggers currently using this item require only Blob host storage.

Set the required `package` field to the full HTTPS download URL of a ready-to-run ZIP in the current space's private GitLab Generic Package registry. Optionally set `packageSha256` to its 64-character lowercase SHA-256 digest when integrity verification is required. A composed Job reads the registry credentials distributed to each node by `registry-operator`, verifies the download when a digest is supplied, always validates the ZIP structure, and deploys it with Azure CLI through the cluster's existing managed identity. Change the immutable URL and optional digest to roll out a new package.

After an automatic package deployment, the connection Secret also contains `function-key` (the default callable host key) and `function-keys` (all callable host keys as JSON). The composition refetches them once per UTC day so externally rotated keys propagate to the Secret. Azure's privileged master and system keys are intentionally excluded.

Use `managedIdentityRoleAssignments` when the Function App must access an existing Azure resource. The catalog limits this field to approved built-in roles and Crossplane creates each assignment only after Azure reports the Function App's system-assigned identity. Scope assignments to the exact target resource; the current approved role manages an AKS cluster. For an existing Key Vault that uses legacy access policies, programmatic clients can set `keyVaultAccessPolicy.keyVaultId` and `keyVaultAccessPolicy.tenantId`; these fields are intentionally hidden from the catalog form, must be supplied together, and grant only the `Get` secret permission needed by App Service Key Vault references. Both access configurations are optional: omitting them leaves the preconfigured system identity and dedicated storage setup unchanged.

See [`examples/azurefunctionflexconsumption.yaml`](examples/azurefunctionflexconsumption.yaml) for a minimal function.

## Azure Storage Account Items (e.g. BlobStorage)

IF you want to add e.g. queue storage, copy paste blob storage and adjust it slightly, see table [here](https://learn.microsoft.com/en-us/azure/private-link/availability#storage)

In order not to expose the storage accoutns publicly, public access is disabled and they are exposed via private endpoint, see [here](https://learn.microsoft.com/en-us/azure/private-link/tutorial-private-endpoint-storage-portal?tabs=dynamic-ip)
