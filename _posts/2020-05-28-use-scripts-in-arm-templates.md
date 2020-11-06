---
layout: post
title: "Execute bash scripts in ARM templates (preview)"
date: 2020-05-28
categories: ["ARM templates"]
---

![infra](/images/2020-05-28-logo.png)

I was recently working with ARM template to provision Azure Database for PostgreSQL and as part of the configuring PostgreSQL instance you need to specify admin password. Normally this task is implemented as a hybrid of initialization script that generates the password and stores it to the key-vault and ARM templates that uses keyvault reference to read password from the key-vault.

When I was preparing labs for my [ARM templates workshop](https://www.meetup.com/Infrastructure-As-Code-User-Group-Oslo/events/270624160/), I came across new ARM templates feature called [Use deployment scripts in templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template?tabs=CLI#run-script-more-than-once).
It allows you to execute scripts (both PowerShell and bash) in ARM templates. It's still in preview, but I decided to give it a try.

## POC requirements

My goal is to implement ARM templates that will:

* generate password using `openssl rand` command
* store password as key-vault secret

## How it works

Behind the scene, script service creates storage account and container instance.

![pic](/images/2020-05-28-resources.png)

* Container instances - that's where your script will be executed
* Storage account - that's where your script will be stored at File shares and used by container instance

Here is a quote from the [documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template?tabs=CLI#clean-up-deployment-script-resources)

> A storage account and a container instance are needed for script execution and troubleshooting. You have the options to specify an existing storage account, otherwise a storage account along with a container instance are automatically created by the script service. The two automatically created resources are deleted by the script service when the deployment script execution gets in a terminal state. You are billed for the resources until the resources are deleted.

Storage account contains the following folders inside `File shares`:

![pic](/images/2020-05-28-sa.png)

*azscriptinput* folder contains 2 scripts

![pic](/images/2020-05-28-sa-1.png)

`userscript.sh` - is where your script from ARM template will be extracted and stored

*azscriptoutput* folder contains `executionresult.json` with script output.

## Prerequisites

First, create new resource group

```bash
az group create -n iac-scriptarm-rg -l westeurope
```

Create key-vault for sensitive data. Keyvault name should be globally unique, therefore you need to define your own key-vault name.

```bash
az keyvault create -n YOUR-KEYVAULT-NAME -g iac-scriptarm-rg -l westeurope --enabled-for-template-deployment
```

### User-assigned managed identity

You need user-assigned identity that will be used to execute deployment scripts. Since script service creates storage account and container instance behind the scene, this managed identity needs a contributor's role at the subscription level.
Since we will store secret to the key-vault, managed identity should have `secret set` permission at the key-vault.

To create identity, use the following script:

```bash
echo -e "Create user-assigned managed identity"
objectId=$(az identity create -n iac-script-arm-mi -g iac-scriptarm-rg --query principalId -o tsv)

echo -e "Assign Contributor role at the subscription level"
az role assignment create --assignee-object-id "$objectId" --role Contributor --subscription YOUR-SUBSCRIPTION-NAME-OR-ID

echo -e "Allow managed identity to read secrets from the key-vault"
az keyvault set-policy -n YOUR-KEYVAULT-NAME --object-id $objectId --secret-permissions get set --subscription YOUR-SUBSCRIPTION-NAME-OR-ID
```

## ARM template

The ARM template contains `Microsoft.Resources/deploymentScripts` resource with inline bash script

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "userAssignedManagedIdentityName": {
            "type": "string",
            "defaultValue": "iac-script-arm-mi"
        }
    },
    "variables": {
        "userAssignedManagedIdentityId": "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('userAssignedManagedIdentityName'))]"
    },
    "resources": [
        {
            "type": "Microsoft.Resources/deploymentScripts",
            "apiVersion": "2019-10-01-preview",
            "name": "generate-and-store-admin-password",
            "location": "[resourceGroup().location]",
            "kind": "AzureCLI",
            "identity": {
                "type": "userAssigned",
                "userAssignedIdentities": {
                    "[variables('userAssignedManagedIdentityId')]": {
                    }
                }
            },
            "properties": {
                "azCliVersion": "2.0.80",
                "scriptContent": "
                password=$(openssl rand -base64 14)
                az keyvault secret set --vault-name YOUR-KEYVAULT-NAME -n admin-password --value $password 1> /dev/null
                ",
                "timeout": "PT30M",
                "cleanupPreference": "OnSuccess",
                "retentionInterval": "P1D"
            }
        }
    ]
}
```

The user-assigned identity has to be specified and `userAssignedManagedIdentityId` is defined as variable

```json
"userAssignedManagedIdentityId": "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('userAssignedManagedIdentityName'))]
```

where `userAssignedManagedIdentityName` passed an ARM template parameter.

The inline script is placed inside `scriptContent` section and contains the following logic.

```bash
password=$(openssl rand -base64 14)
az keyvault secret set --vault-name YOUR-KEYVAULT-NAME -n admin-password --value $password 1> /dev/null
```

## Deploy script

Now we can deploy our script by running the following command:

```bash
az deployment group create -g iac-scriptarm-rg --template-file template.json
```

and check that password was generated and stored at the key-vault:

```bash
az keyvault secret show -n admin-password --vault-name YOUR-KEYVAULT-NAME --query value
```

## `retentionInterval` and `cleanupPreference` parameters

`cleanupPreference` controls when automatically created resources (storage account and container instance) should be deleted. The supported values are `Always`, `OnSuccess` and `OnExpiration`. Read more about [Clean up deployment script resources](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template?tabs=CLI#clean-up-deployment-script-resources)

`retentionInterval` specifies the time interval that a script resource will be retained and after which will be expired and deleted.

### How about idempotency

By default, the deployment script service compares the resource names in the template with the existing resources in the same resource group. If the names are the same, the script will not be executed. In our scenario, this is exactly behavior we need. We definitely don't want to generate new password every time we run deployment. We want password to be generated once, during first time provisioning.  
At the same time, if you want to execute the same deployment script multiple times, there are 2 options:

* Change the name of your deploymentScripts resource
* Specify a different value in the `forceUpdateTag` template property

Read more about idempotency [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template?tabs=CLI#run-script-more-than-once).

## To sum-up

I have a mixed feelings. I definitely like the idea and I can see a lot of use-cases where I will use it, but at the same time, I don't like inline scripts as part of the json file. There is an option to [use external scripts](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template?tabs=CLI#use-external-scripts), but in this case scrips either have to be publicly available (for example via github), or you have to upload them to storage account and use sas token to access them and that increases the complexity of already-not-that-easy-to-work-with ARM templates.

## Don't forget to remove your resources

Clean up your resource by running the following command:

```bash
az group delete -n iac-scriptarm-rg
```

## Useful links

* [Use deployment scripts in templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template?tabs=CLI#run-script-more-than-once)
* [Tutorial: Use deployment scripts to create a self-signed certificate (Preview)](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-tutorial-deployment-script)
* [Microsoft.Resources deploymentScripts template reference](https://docs.microsoft.com/en-us/azure/templates/microsoft.resources/deploymentscripts)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
