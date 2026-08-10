# Azure RBAC from Crossplane Compositions

Research date: 7 August 2026

## Conclusion

Crossplane can manage these grants without Azure Function-specific operator code. The supported cloud resource is the namespaced `RoleAssignment` managed resource from Upbound's `provider-azure-authorization`. A composition can render it after the principal ID and target Azure resource ID appear in the observed status of the composed Function App and storage account.

This is the strongest implementation option if PlatformPlane accepts delegating constrained Azure role-assignment permission to a Crossplane credential. It is not permission-free: the credential used by `provider-azure-authorization` must already have `Microsoft.Authorization/roleAssignments/write` (and `delete` for normal cleanup) at the relevant scope. Azure `Contributor` explicitly cannot assign roles. That bootstrap permission must therefore be granted outside the composition, for example by the cluster operator or a platform administrator.

If that extra Crossplane privilege is unacceptable, the current privileged operator remains necessary. There is no Crossplane core permission-request/grant API that delegates the Azure operation back to an external privileged controller.

## Native provider mechanism

The current Azure authorization provider exposes a namespace-scoped `authorization.azure.m.upbound.io/v1beta1` `RoleAssignment`. Its inputs include `principalId`, `principalType`, `roleDefinitionId` or `roleDefinitionName`, and `scope`; it also accepts a `providerConfigRef`. Upbound describes the provider as assigning RBAC roles at subscription, resource-group, and resource scope to users, groups, and managed identities.

Sources:

- [Upbound provider-azure-authorization v2.6.0](https://marketplace.upbound.io/providers/upbound/provider-azure-authorization/v2.6.0)
- [RoleAssignment v1beta1 schema](https://marketplace.upbound.io/providers/upbound/provider-azure-authorization/v2.0.1/resources/authorization.azure.m.upbound.io/RoleAssignment/v1beta1)
- [Provider source repository](https://github.com/crossplane-contrib/provider-upjet-azure)

The schema has a reference/selector for `roleDefinitionId`, but not for `principalId` or `scope`. The provider maintainers closed a request for `principalIdRef` as not planned and recommended resolving the value in a Composition. Therefore a plain provider cross-resource reference is not enough for the Function-to-storage relationship.

Source: [provider-upjet-azure issue 672](https://github.com/crossplane-contrib/provider-upjet-azure/issues/672)

## How a composition resolves the IDs

The catalog already uses `function-go-templating`. That function receives observed composed resources and supports reading fields such as `status.atProvider`; it can conditionally emit the `RoleAssignment` only after both values exist. In this case the same Azure Function composition already owns the Function App and either creates or selects storage, so this is simpler than adding another controller.

For a separately composed or previously existing resource, `function-go-templating` can emit an `ExtraResources` request by name or labels. Crossplane fetches the matching local Kubernetes resources and provides them to the template. Crossplane v2 also supports required resources in a pipeline step and dynamic resource requests, although dynamic requests are limited to five iterations. The template must still fail closed if selection is missing or ambiguous and validate namespace, catalog ownership labels, Azure resource type, subscription, and resource group before using a returned ID.

Sources:

- [function-go-templating: observed resources and ExtraResources](https://github.com/crossplane-contrib/function-go-templating#extraresources)
- [Crossplane compositions: required resources](https://docs.crossplane.io/latest/composition/compositions/#required-resources)

`function-patch-and-transform` can template the `RoleAssignment`, but it cannot directly patch one composed resource into another. It requires using composite status as an intermediate field, which makes it less suitable than the Go template already present. `function-cel-filter` only filters desired resources; it does not resolve Azure identities or perform grants.

Sources:

- [Function Patch and Transform: patching between resources](https://docs.crossplane.io/latest/guides/function-patch-and-transform/#patching-between-resources)
- [function-cel-filter](https://github.com/crossplane-contrib/function-cel-filter)

## Sequencing and deletion

No imperative wait loop is required. Composition functions are called repeatedly with updated observed state, so the template can omit the `RoleAssignment` until the Function principal and storage scope exist. `function-sequencer` can additionally delay the role assignment until the earlier composed resources report ready.

Crossplane `Usage` and the sequencer's deletion sequencing are not authorization grants. `Usage` only protects resources from deletion and establishes deletion order. It may be useful to delete role assignments before storage or Function resources, but it cannot create Azure RBAC access.

Sources:

- [function-sequencer](https://github.com/crossplane-contrib/function-sequencer)
- [Crossplane Usages](https://docs.crossplane.io/latest/managed-resources/usages/)

## Why provider-kubernetes does not help

`provider-kubernetes/Object` can create arbitrary Kubernetes objects in another configured Kubernetes API and can patch fields from referenced objects. Wrapping a same-cluster Azure `RoleAssignment` in an `Object` adds another provider, credential, and reconciliation layer while the composition can already emit the managed resource directly. It does not remove the Azure privilege requirement from `provider-azure-authorization`.

Source: [provider-kubernetes Object v1alpha1](https://marketplace.upbound.io/providers/crossplane-contrib/provider-kubernetes/v1.2.1/resources/kubernetes.m.crossplane.io/Object/v1alpha1)

## Credential and security boundary

Azure requires `Microsoft.Authorization/roleAssignments/write` to create assignments. `Role Based Access Control Administrator` and `User Access Administrator` have that permission; `Contributor` does not. Microsoft recommends the smallest possible scope. A bootstrap assignment at the PlatformPlane resource-group scope limits the provider to that resource group and its children.

Azure also supports conditions on the delegate's administrator assignment. PlatformPlane can restrict the authorization credential to:

- only the approved storage data-role definition IDs;
- only `ServicePrincipal` principals, which covers managed identities;
- create and delete role assignments only inside the PlatformPlane resource group.

The create and delete paths need matching conditions because Azure evaluates request attributes for creation and resource attributes for deletion. These conditions reduce the blast radius, but they still let the credential assign the allowlisted roles to any service principal at an allowed scope. They do not reproduce the current operator's catalog-owner validation.

Sources:

- [Steps to assign an Azure role](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-steps)
- [Delegate Azure role assignment management with conditions](https://learn.microsoft.com/en-us/azure/role-based-access-control/delegate-role-assignments-portal)
- [Delegation condition examples](https://learn.microsoft.com/en-us/azure/role-based-access-control/delegate-role-assignments-examples)
- [Azure role-assignment scope guidance](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments#scope)

The Azure provider family supplies namespaced `ProviderConfig` and cluster-scoped `ClusterProviderConfig` credentials. A dedicated provider config and dedicated Azure identity for authorization work is preferable to adding RBAC-administrator permission to the existing general `default` credential. Kubernetes RBAC must prevent catalog users from creating arbitrary `RoleAssignment` managed resources or selecting this privileged provider config directly.

Sources:

- [provider-family-azure](https://marketplace.upbound.io/providers/upbound/provider-family-azure/v2.6.1)
- [ClusterProviderConfig schema](https://marketplace.upbound.io/providers/upbound/provider-family-azure/v2.5.3/resources/azure.m.upbound.io/ClusterProviderConfig)

## Recommendation

1. Add `provider-azure-authorization` to the catalog configuration.
2. Use a dedicated Azure identity and provider config for authorization, not the general `default` provider config.
3. Bootstrap that identity once with conditional `Role Based Access Control Administrator` at only the PlatformPlane resource-group scope, allowlisting the storage roles and `ServicePrincipal` principals.
4. Have the Azure Function composition emit namespaced `RoleAssignment` resources only when the observed Function principal ID and validated storage resource ID are present.
5. Keep role choice in the catalog composition. Do not expose raw role IDs, principal IDs, scopes, or the privileged provider-config name in the user-facing Azure Function API.
6. Remove the Azure Function-specific role scanner from `crossplane-azure-operator` only after an integration test proves creation and cleanup through the authorization provider.

For future services, repeat the same composition pattern or introduce an internal, non-user-creatable `AzureAccessGrant` composite API with allowlisted roles and trusted resource references. Crossplane does not provide that generic cloud grant abstraction itself; it would be a PlatformPlane API backed by the same `RoleAssignment` managed resource.

## Implementation outcome

Implemented with two deliberate deviations from the recommendation above, both to avoid re-coupling `crossplane-azure-operator` to Azure Function/Storage specifics as new catalog items reuse this path:

- No second identity or dedicated provider config (recommendation #2). The bootstrap grant was added to the same per-cluster identity that already holds `Contributor`, as a second, conditioned role assignment. A separate identity only helps if a composition would otherwise have to be trusted with an unconditioned grant; since the condition is what actually bounds the blast radius, a second identity mainly buys an extra opt-in step, not additional safety.
- The delegation condition denylists the roles that grant further delegation (`Owner`, `Role Based Access Control Administrator`, `User Access Administrator`) and restricts principals to `ServicePrincipal`, rather than allowlisting the three storage roles. An allowlist would need editing in `crossplane-azure-operator` every time a future catalog item needs a *different* role grant - exactly the coupling this work removes. See `crossplane-azure-operator/pkg/operator/operator_helper_rbac.go`.

`AzureFunction`'s storage was also made always-dedicated (no `existingStorageName` reuse) and composed inline rather than through a nested `AzureStorageV2` XR, so the Function's own composition pipeline can read the account's resolved resource ID directly from `.observed.resources` without a cross-composition lookup (recommendation #4 was going to need `ExtraResources` for the reuse case; removing reuse removed the need for it). The Azure Function-specific role scanner in `crossplane-azure-operator` was removed in the same change as the composition switch (recommendation #6 suggested removing it only after independent validation of creation and cleanup) - accepted here as a known, explicit deviation.
