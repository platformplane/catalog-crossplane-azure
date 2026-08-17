# Azure Crossplane Self-Service Catalog Items

Azure-specific items for the Crossplane Service Catalog. See [`catalog-crossplane`](https://github.com/platformplane/catalog-crossplane) for general documentation.

## Azure Function (Flex Consumption)

`AzureFunctionFlexConsumption` creates a Linux Function App on a Flex Consumption (`FC1`) plan. Custom container images are not supported.

## Azure Storage Accounts

`AzureStorageV2` creates private endpoints for Blob, File, Queue, and Table storage. Keep `public` disabled to prevent public network access.
