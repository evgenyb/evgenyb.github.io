---
layout: post
title: "ARM template for AKS cluster with managed identity and managed Azure AD integration"
date: 2020-08-05
categories: ["AKS", "Managed Identity", "Azure AD", "ARM templates", "Infrastructure as Code"]
---

Recently I was working with my sandbox infrastructure environment and decided to switch my AKS cluster to [use Managed Identity](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity?WT.mc_id=AZ-MVP-5003837) and [managed Azure AD integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad?WT.mc_id=AZ-MVP-5003837). I mainly use ARM templates to describe my infrastructure and I actually didn't find any ARM template samples showing how to configure AKS with neither managed identity, nor managed Azure AD integration, so I decided to share what I finally came up with as a solution...

Here are my AKS cluster configuration requirements:

* AKS is deployed to `iac-aks-blue|green-rg` resource group
* AKS is called `iac-blue|green-aks`
* AKS is deployed to `aks-net` subnet of `iac-aks-blue|green-vnet` private virtual network
* AKS uses [advanced networking](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni?WT.mc_id=AZ-MVP-5003837)
* AKS uses [Calico networking policies](https://docs.microsoft.com/en-us/azure/aks/use-network-policies?WT.mc_id=AZ-MVP-5003837)
* AKS uses pre-provisioned egress public IP address called `iac-aks-blue|green-egrees-pip` (also part of the ARM templates)
* AKS uses managed identity
* AKS uses `iac-admin` Azure AD group for managed Azure AD integration

## AKS Managed Identity and role assignment

For resources outside of the AKS "managed" `MC_*` resource group, AKS managed identity needs to be granted with required permissions, so AKS is able to interact with “external” resources (for example, read/write on subnets or provision static IP address etc.). AKS managed identity has to be assigned with `NetworkContributor` role at the AKS subnet scope. To perform a role assignment, use the `principalId` of the cluster System Assigned managed identity. Here is an example how you can assign `NetworkContributor` role (you can find role GUID in [Azure built-in roles](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles?WT.mc_id=AZ-MVP-5003837) list) for AKS managed identity with ARM template.

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

AKS-managed Azure AD integration is designed to simplify the Azure AD integration experience, where users were previously required to create a client app, a server app, and required the Azure AD tenant to grant Directory Read permissions. In the new version, the AKS resource provider manages the client and server apps for you.

To enable Azure AD integration, add the `aadProfile` property inside `properties` section and use the following syntax:

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
az ad group show -g 'iac-admin' --query objectId
```

## apiVersion

The solution shown above works only when I set `apiVersion` of `Microsoft.ContainerService/managedClusters` to `2020-06-01` (and you actually get this version if you create your cluster with `az cli` or from portal and then export AKS template), but I didn't find this version in the "official" [Microsoft.ContainerService managedClusters template reference](https://docs.microsoft.com/en-us/azure/templates/microsoft.containerservice/managedclusters?WT.mc_id=AZ-MVP-5003837).

## Final version

Here is the complete version of my [ARM template](https://github.com/evgenyb/arm/blob/master/aks/).

## Useful links

* [Use managed identities in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity?WT.mc_id=AZ-MVP-5003837)
* [AKS-managed Azure Active Directory integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad?WT.mc_id=AZ-MVP-5003837)
* [Azure built-in roles](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles?WT.mc_id=AZ-MVP-5003837)
* [Microsoft.ContainerService managedClusters template reference](https://docs.microsoft.com/en-us/azure/templates/microsoft.containerservice/managedclusters?WT.mc_id=AZ-MVP-5003837)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
