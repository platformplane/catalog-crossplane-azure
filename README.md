# Azure Crossplane Self-Service Catalog Items

This repo contains the Crossplane Service Catalog items that are specific to Azure.

For more information, have a look at the [catalog-crossplane](https://github.com/platformplane/catalog-crossplane) repository.

## Azure Function

`AzureFunction` provisions a containerized Linux Function App in Switzerland North on an Elastic Premium plan. Set the tagged image as one `registry/name:tag` value, for example `registry.example.com/team/function:tag`, and add any non-secret environment variables. Premium plans keep a billed instance available even when the function is idle.

Each function always gets its own dedicated, private storage account for host storage, composed directly by this item (it is not a separate `AzureStorageV2` catalog resource, so there is nothing to share or reuse). Host storage uses the Function App's system-assigned managed identity instead of an account key: once the identity and the account are both observed, the composition grants that identity the Blob Data Owner, Queue Data Contributor, and Table Data Contributor roles on its own account via a Crossplane-managed `RoleAssignment` - no operator code is involved. Azure role propagation can delay the first successful Function start. Azure Files content storage is disabled for this Linux Premium container deployment, so no file private endpoint is created either.

Private registries are supported by setting `registryCredentialsSecretName` to a Secret in the same namespace with `username` and `password` keys. The resulting app name, URL, and hostname are written to `<resource-name>-connection` in the resource namespace.

See [`examples/azurefunction.yaml`](examples/azurefunction.yaml) for a minimal function.

## Azure Function (Flex Consumption)

`AzureFunctionFlexConsumption` provisions a Linux Function App on a Flex Consumption (`FC1`) plan instead of Elastic Premium: fast, pay-per-use elastic scaling, with no billed instance kept warm while idle. Flex Consumption does not support custom container images in the provider this catalog uses - set `runtimeName`/`runtimeVersion` (e.g. `node`/`20`) instead of an image, and add any non-secret environment variables. Use `AzureFunction` instead if you need to deploy an arbitrary OCI container image.

Each function gets its own dedicated, private storage account holding just the deployment package container (Flex Consumption has no separate queue/table host-storage usage the way Premium plans do). The Function App's system-assigned managed identity is granted Blob Data Owner on that one container via a Crossplane-managed `RoleAssignment`, scoped to the container rather than the whole account.

See [`examples/azurefunctionflexconsumption.yaml`](examples/azurefunctionflexconsumption.yaml) for a minimal function.

## Azure Storage Account Items (e.g. BlobStorage)

IF you want to add e.g. queue storage, copy paste blob storage and adjust it slightly, see table [here](https://learn.microsoft.com/en-us/azure/private-link/availability#storage)

In order not to expose the storage accoutns publicly, public access is disabled and they are exposed via private endpoint, see [here](https://learn.microsoft.com/en-us/azure/private-link/tutorial-private-endpoint-storage-portal?tabs=dynamic-ip)
