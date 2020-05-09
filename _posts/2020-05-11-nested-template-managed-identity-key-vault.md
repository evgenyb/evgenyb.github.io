---
layout: post
title: "Use nested ARM template to provision User Assigned Managed Identity and add access policy to key-vault from different subscription"
date: 2020-05-11
categories: [ARM templates]
---

Azure Application Gateway v2 supports integration with Key Vault for server certificates that are attached to HTTPS-enabled listeners. One of the main benefits of this integration, is support for automatic renewal of certificates. This is especially practical if you use Azure App Service Certificates that you purchase and maintain directly on Azure.

Application Gateway integration with key-vault requires a three-step configuration process:

![infra](/images/2020-05-11_1305.png)

### 1. Create a user-assigned [managed identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)

### 2. Configure access policy at key-vault

We need to define access policies in the key-vault to allow the identity to be granted get access to the secret.

### 3. Configure the application gateway

After we complete the two previous steps, we can configure application gateway to use the user-assigned managed identity

```json
"identity": {
    "type": "UserAssigned",
    "userAssignedIdentities": {
        "managed-identity-id": {
        }
    }
}
```

where `userAssignedIdentities` is defined as

> The list of user identities associated with resource. The user identity dictionary key references will be ARM resource ids in the form: '/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{identityName}'.

And  HTTP listener’s TLS/SSL certificate to point to the complete URI of secret ID containing certificate.

```json
"sslCertificates": [
    {
        "name": "appGwSslCertificate",
        "properties": {
            "keyVaultSecretId": "[parameters('certificateKeyVaultSecretId')]"
        }
    }
]
```

Check out this complete implementation of ARM template for [AGW v2 with key-vault integration](https://github.com/Azure/azure-quickstart-templates/tree/master/101-application-gateway-key-vault-create) from [Azure Resource Manager QuickStart Templates collection](https://github.com/Azure/azure-quickstart-templates).

At my current project we use "hybrid" approach, that is - we provision Application Gateway instance using ARM templates, but creation of the user assigned managed identities and configuring access policy at key-vault we do with `az cli` script as part of resource group initialization process. Here is the script that does the job.

```bash
#!/usr/bin/env bash
# Usage:
#   ./02-create-mi.sh iac-certificates-kv <kv-subscription-id> iac-agw-mi iac-agw-ssl-rg

kvName=$1
kvSubscriptionId=$2
miName=$3
miResourceGroupName=$4

echo -e "Create new user assigned managed identity ${miName} in ${miResourceGroupName}"
miObjectId=$(az identity create -n ${miName} -g ${miResourceGroupName} -l westeurope --query principalId -o tsv)

echo -e "Grant ${miName}(${miObjectId}) secret get permission at ${kvName}"
az keyvault set-policy --name ${kvName} --object-id ${miObjectId} --secret-permissions get --subscription ${kvSubscriptionId} 1> /dev/null
```

In this post I decided to check what it takes to implement ARM template that does the same.

## Use-case description

The key-vault containing SSL certificates is located at the different Azure subscription. What we want to implement is ARM template that will:

* create user assigned managed identity called `iac-agw-mi`
* grant `iac-agw-mi` managed identity `get` access policy to the secrets level at `iac-certificates-kv` key-vault.

## User Assigned Managed Identity

This is probably one of the simplest ARM resources you can find. Here is [Microsoft.ManagedIdentity userAssignedIdentities template reference documentation](https://docs.microsoft.com/en-us/azure/templates/microsoft.managedidentity/2018-11-30/userassignedidentities). There are no configurable properties, so the resource implementation looks like this:

```json
{
    "name": "[parameters('managedIdentityName')]",
    "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
    "apiVersion": "2018-11-30",
    "location": "[variables('loaction')]"
}
```

## Configure key-vault accessPolicies

You can manage key-vault's access policies with [Microsoft.KeyVault vaults/accessPolicies](https://docs.microsoft.com/en-us/azure/templates/microsoft.keyvault/2019-09-01/vaults/accesspolicies) resource.

Here is one good [example](https://github.com/Azure/azure-quickstart-templates/blob/master/101-keyvault-add-access-policy/azuredeploy.json) of how you can add an Access Policy to an existing key-vault.

Let's look at the syntax:

```json
{
    "type": "Microsoft.KeyVault/vaults/accessPolicies",
    "name": "[concat(parameters('certificateKeyVaultName'), '/add')]",
    "apiVersion": "2019-09-01",
    "properties": {
        "accessPolicies": [
            {
                "tenantId": "[parameters('tenantId')]",
                "objectId": "[reference(parameters('managedIdentityName')).principalId]",
                "permissions": {
                    "secrets": [
                        "get"
                    ]
                }
            }
        ]
    }
}
```

`name` can be one of the following values (add, replace, remove). Since we adding the new policy, we will use `/add`.

`objectId` - is the object ID of our managed identity, since both resources are at the same ARM template, we can use [reference()](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#reference) function to get our managed identity object id.

`tenantId` - the Azure Active Directory tenant ID that should be used for authenticating requests to the key vault.

`permissions` - permissions the identity has for keys, secrets and certificates.

If we try to deploy this template, it will fail with the following error message:

```bash
$ az deployment group create -g iac-agw-ssl-rg --template-file template.json
Deployment failed. Correlation ID: xxxxxxxxxxxxxxx. {
  "error": {
    "code": "ParentResourceNotFound",
    "message": "Can not perform requested operation on nested resource. Parent resource 'iac-certificates-kv' not found."
  }
}
```

That's actually correct, because our certificate key-vault (`iac-certificates-kv`) is not located at the `iac-agw-ssl-rg` resource group. It's deployed to the `domains-and-certificates` resource group located at different subscription. How can we solve this with ARM templates?

## Nested templates

ARM templates has a technique called, [nested templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates#nested-template).

To nest a template, add a [deployments resource](https://docs.microsoft.com/en-us/azure/templates/microsoft.resources/2019-10-01/deployments) to your main template. In the `template` property, specify the ARM template syntax:

```json
{
    "name": "grant-access",
    "type": "Microsoft.Resources/deployments",
    "apiVersion": "2019-10-01",
    "subscriptionId": "[parameters('certificateKeyVaultSubscriptionId')]",
    "resourceGroup": "[parameters('certificateKeyVaultResourceGroupName')]",
    "dependsOn": [
        "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('managedIdentityName'))]"
    ],
    "properties": {
        "mode": "Incremental",
        "template": {
            "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
                {
                    "type": "Microsoft.KeyVault/vaults/accessPolicies",
                    "name": "[concat(parameters('certificateKeyVaultName'), '/add')]",
                    "apiVersion": "2019-09-01",
                    "properties": {
                        "accessPolicies": [
                            {
                                "tenantId": "[parameters('tenantId')]",
                                "objectId": "[reference(parameters('managedIdentityName')).principalId]",
                                "permissions": {
                                    "secrets": [
                                        "get"
                                    ]
                                }
                            }
                        ]
                    }
                }
            ]
        }
    }
}
```

As you can see, we can specify what subscription (`subscriptionId` attribute) and resource group (`resourceGroup` attribute) we want to deploy resource to. Inside `properties->template` element, we just implemented regular `Microsoft.KeyVault/vaults/accessPolicies` resource.

## Final version

Here is our final version of the template:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "certificateKeyVaultName": {
            "type": "string"
        },
        "tenantId": {
            "type": "string",
            "defaultValue": "[subscription().tenantId]"
        },
        "certificateKeyVaultResourceGroupName": {
            "type": "string"
        },
        "certificateKeyVaultSubscriptionId": {
            "type": "string"
        },
        "managedIdentityName": {
            "type": "string"
        }
    },
    "variables": {
        "loaction": "[resourceGroup().location]"
    },
    "resources": [
        {
            "name": "[parameters('managedIdentityName')]",
            "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
            "apiVersion": "2018-11-30",
            "location": "[variables('loaction')]"
        },
        {
            "name": "grant-access",
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2019-10-01",
            "subscriptionId": "[parameters('certificateKeyVaultSubscriptionId')]",
            "resourceGroup": "[parameters('certificateKeyVaultResourceGroupName')]",
            "dependsOn": [
                "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('managedIdentityName'))]"
            ],
            "properties": {
                "mode": "Incremental",
                "template": {
                    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
                    "contentVersion": "1.0.0.0",
                    "resources": [
                        {
                            "type": "Microsoft.KeyVault/vaults/accessPolicies",
                            "name": "[concat(parameters('certificateKeyVaultName'), '/add')]",
                            "apiVersion": "2019-09-01",
                            "properties": {
                                "accessPolicies": [
                                    {
                                        "tenantId": "[parameters('tenantId')]",
                                        "objectId": "[reference(parameters('managedIdentityName')).principalId]",
                                        "permissions": {
                                            "secrets": [
                                                "get"
                                            ]
                                        }
                                    }
                                ]
                            }
                        }
                    ]
                }
            }
        }
    ]
}
```

## To sum-up

So, I managed to grant access to key-vault for user assigned identity from ARM templates. I personally think that it's easier with `az cli` script, but at the same time, if you don't want to (or if you are not allowed to) use a mix of ARM templates and `az cli` scripts, it's totally possible to implement everything with ARM templates.

## Useful links

* [TLS termination with Key Vault certificates](https://docs.microsoft.com/en-us/azure/application-gateway/key-vault-certs)
* [Purchase certificate](https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate#start-certificate-order)
* [What are managed identities for Azure resources?](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
* [Microsoft.Network applicationGateways template reference](https://docs.microsoft.com/en-us/azure/templates/microsoft.network/2020-04-01/applicationgateways)
* [ARM templates Nested templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates#nested-template)
* [ARM template reference() function](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#reference)
* [add an Access Policy to an Existing Keyvault example](https://github.com/Azure/azure-quickstart-templates/blob/master/101-keyvault-add-access-policy/azuredeploy.json)
* [Azure Resource Manager QuickStart Templates collection](https://github.com/Azure/azure-quickstart-templates)
* [Microsoft.KeyVault vaults/accessPolicies template reference documentation](https://docs.microsoft.com/en-us/azure/templates/microsoft.keyvault/2019-09-01/vaults/accesspolicies)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
