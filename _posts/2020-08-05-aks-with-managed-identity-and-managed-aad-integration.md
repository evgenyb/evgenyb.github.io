---
layout: post
title: "ARM template for AKS cluster with managed identity and managed Azure AD integration"
date: 2020-08-05
categories: [Azure, AKS, Managed Identity, Azure AD, ARM templates, Kubernetes, Infrastructure as Code]
---

Recently I was working with my sandbox infrastructure environment and decided to switch my AKS cluster to [use Managed Identity](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity) and [managed Azure AD integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad). I mainly use ARM templates to describe my infrastructure and I actually didn't find any ARM template samples showing how to configure AKS with neither managed identity, nor managed Azure AD integration, so I decided to share what I finally came up with as a solution...

Here are my AKS cluster configuration requirements:

* AKS is deployed to `iac-aks-blue|green-rg` resource group
* AKS is called `iac-blue|green-aks`
* AKS is deployed to `aks-net` subnet of `iac-aks-blue|green-vnet` private virtual network
* AKS uses [advanced networking](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)
* AKS uses [Calico networking policies](https://docs.microsoft.com/en-us/azure/aks/use-network-policies)
* AKS uses pre-provisioned egress public IP address called `iac-aks-blue|green-egrees-pip` (also part of the ARM templates)
* AKS uses managed identity
* AKS uses `iac-admin` Azure AD group for managed Azure AD integration

## AKS Managed Identity and role assignment

For resources outside of the AKS "managed" `MC_*` resource group, AKS managed identity needs to be granted with required permissions, so AKS is able to interact with “external” resources (for example, read/write on subnets or provision static IP address etc.). AKS managed identity has to be assigned with `NetworkContributor` role at the AKS subnet scope. To perform a role assignment, use the PrincipalID of the cluster System Assigned managed identity. Here is an example how you can assign `NetworkContributor` role (you can find role GUID in [Azure built-in roles](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles) list) for AKS managed identity with ARM template.

```json
{
    "type": "Microsoft.Network/virtualNetworks/subnets/providers/roleAssignments",
    "apiVersion": "2017-05-01",
    "name": "[concat(parameters('vnetName'), '/', parameters('subnetName'), '/Microsoft.Authorization/', guid(resourceGroup().id, 'akstovnet'))]",
    "properties": {
        "roleDefinitionId": "[variables('networkContributorRole')]",
        "principalId": "[reference(resourceId('Microsoft.ContainerService/managedClusters/', parameters('clusterName')), '2020-06-01', 'Full').identity.principalId]",
        "scope": "[variables('subnetId')]"
    }
}
```

To enable system-assigned managed identity, add the `identity` property at the same level as the "type": "Microsoft.ContainerService/managedClusters" property. Use the following syntax:

```json
"identity": {
    "type": "SystemAssigned"
}
```

and then set `clientId` field of `servicePrincipalProfile` property to `msi`  

```json
"servicePrincipalProfile": {
    "clientId": "msi"
}
```

## Azure AD integration

To enable Azure AD integration, add the `aadProfile` property inside `properties` section. Use the following syntax:

```json
"aadProfile": {
    "managed": true,
    "tenantId": "[parameters('tenantId')]",
    "adminGroupObjectIDs": [
        "[parameters('adminGroupObjectId')]"
    ]
}
```

To find your Azure AD group id by name, use the following command:

```bash
az ad group  show -g 'iac-admin' --query objectId
```

Here is the complete version of the [ARM template](https://github.com/evgenyb/arm/blob/master/aks/template.json).

## Useful links

* [Use managed identities in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity)
* [AKS-managed Azure Active Directory integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad)
* [Azure built-in roles](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
