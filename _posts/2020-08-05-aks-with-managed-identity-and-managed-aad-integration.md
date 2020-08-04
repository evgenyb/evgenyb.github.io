---
layout: post
title: "ARM template for AKS cluster with managed identity and managed Azure AD integration"
date: 2020-08-05
categories: [Azure, AKS, Managed Identity, Azure AD, ARM templates, Kubernetes, Infrastructure as Code]
---

Recently I was working with my sandbox infrastructure environment and decided to switch my AKS cluster to [use Managed Identity](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity) and [managed Azure AD integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad). I mainly use ARM templates to describe my infrastructure and I actually didn't find any ARM template samples showing how to configure AKS with neither managed identity, nor managed Azure AD integration, so I decided to share what I finally came up with as a solution...

Here are my AKS cluster configuration requirements:

* AKS is deployed to `iac-aks-blue|green-rg` resource group
* AKS called `iac-blue|green-aks`
* AKS is deployed to `aks-net` subnet of `iac-aks-blue|green-vnet` private virtual network
* AKS uses [advanced networking](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)
* AKS uses [Calico networking policies](https://docs.microsoft.com/en-us/azure/aks/use-network-policies)
* AKS uses pre-provisioned egress public IP address called `iac-aks-blue|green-egrees-pip` (also part of the ARM templates)
* AKS uses managed identity
* AKS uses `iac-admin` Azure AD group for managed Azure AD integration

## Managed Identities

For resources outside of the AKS "managed" `MC_*` resource group, AKS managed identity needs to be granted with required permissions, so AKS will be able to interact with “external” resources (for example, read/write on subnets, provision static IP address etc.).

Use the PrincipalID of the cluster System Assigned Managed Identity to perform a role assignment.

Here is an example how you can assign `NetworkContributorRole` for AKS managed identity and with ARM template:

```json
{
    "type": "Microsoft.Network/virtualNetworks/subnets/providers/roleAssignments",
    "apiVersion": "2017-05-01",
    "name": "[concat(parameters('vnetName'), '/', parameters('subnetName'), '/Microsoft.Authorization/', guid(resourceGroup().id, 'aksvnet'))]",
    "properties": {
        "roleDefinitionId": "[variables('networkContributorRole')]",
        "principalId": "[reference(resourceId('Microsoft.ContainerService/managedClusters/', parameters('clusterName')), '2020-06-01', 'Full').identity.principalId]",
        "scope": "[variables('subnetId')]"
    }
}
``` 

## Useful links

* [Use managed identities in Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity)
* [AKS-managed Azure Active Directory integration](https://docs.microsoft.com/en-us/azure/aks/managed-aad)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
