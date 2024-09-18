# Azure Crossplane Self-Service Catalog Items

This repo contains the Crossplane Service Catalog items that are specific to Azure.

For more information, have a look at the [catalog-crossplane](https://github.com/platformplane/catalog-crossplane) repository.

## Azure Storage Account Items (e.g. BlobStorage)

IF you want to add e.g. queue storage, copy paste blob storage and adjust it slightly, see table [here](https://learn.microsoft.com/en-us/azure/private-link/availability#storage)

In order not to expose the storage accoutns publicly, public access is disabled and they are exposed via private endpoint, see [here](https://learn.microsoft.com/en-us/azure/private-link/tutorial-private-endpoint-storage-portal?tabs=dynamic-ip)
