---
layout: post
title: "ARM template deployment what-if operation"
date: 2020-04-27
categories: ["ARM templates", "PowerShell"]
---

During the preparation for my ["How to live in harmony with ARM templates" on-line hands-on workshop](https://www.meetup.com/Infrastructure-As-Code-User-Group-Oslo/events/268754221/) I came across the following really interesting [Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview) feature called [what-if operation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-deploy-what-if). This is similar concept to [Terraform Plan](https://www.terraform.io/docs/commands/plan.html), when Resource Manager will compare the current infrastructure state with ARM template you are trying to deploy and show the "execution plan" without making any changes to existing resources. That is - it will show what will be added, removed or updated. So, I decided to check this out and get some hands-on experience how it actually works.

Note, this feature is in preview and only available for PowerShell commands or REST API operations, so you shouldn't use it in your production pipelines.

## Install powershell module(s)

First, you need to install the preview version of [Az.Resources](https://docs.microsoft.com/en-us/powershell/module/az.resources/?view=azps-3.8.0) module from the PowerShell gallery.

```powershell
Install-Module Az.Resources -RequiredVersion 1.12.1-preview -AllowPrerelease
```

If you get an error (I've got an error), you may need to install [PowerShellGet](https://docs.microsoft.com/en-us/powershell/module/powershellget/?view=powershell-7) module

```powershell
Install-Module PowerShellGet -Repository PSGallery -Force
```

## Create new resource group

Next, let's create new resource group. Since `-whatIf` feature is only supported in PowerShell now, let's use PowerShell as a scripting language.

Make sure that you logged in to your azure account

```powershell
Connect-AzAccount
```

then select your active subscription (if you have several subscriptions)

```powershell
Select-AzSubscription -SubscriptionName "your-subscription-name"
```

and finally create new resource group

```powershell
New-AzResourceGroup -Name "iac-what-if-poc-rg" -Location "West Europe"
```

## Create an empty ARM template

Next, let's create an empty ARM template.

Tip. If you use [VS Code](https://code.visualstudio.com/download) then install the [ARM tool plugin](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools)

Then create an empty file called `template.json`. Type `arm!` and `ARM tool` will create an empty but valid ARM template. If you don't use VS code and / or don't want to install the plugin, then just use this json

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "functions": [],
    "variables": {},
    "resources": [],
    "outputs": {}
}
```

Now, let's run test `-WhatIf` command

```powershell
New-AzResourceGroupDeployment -ResourceGroupName iac-what-if-poc-rg -TemplateFile template.json -WhatIf

Note: As What-If is currently in preview, the result may contain false positive predictions (noise).
You can help us improve the accuracy of the result by opening an issue here: https://aka.ms/WhatIfIssues.


Resource changes: no change.
```

Cool, no changes and that's expected. Now, let's add new storage account resource into `resources` section of ARM template file.

Note that I used [uniqueString()](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-string#uniquestring) function here to make sure that storage account has a unique name. Also, storage account max length is 24 characters long and `uniqueString` function returned value is 13 characters long, so you need to do some math :)

```json
{
    "name": "[concat('iacwhatif', uniqueString(resourceGroup().id))]",
    "type": "Microsoft.Storage/storageAccounts",
    "apiVersion": "2019-06-01",
    "location": "[resourceGroup().location]",
    "kind": "StorageV2",
    "sku": {
        "name": "Standard_LRS",
        "tier": "Standard"
        }
}
```

and run `-whatIf` command again...

```powershell
New-AzResourceGroupDeployment -ResourceGroupName iac-what-if-poc-rg -TemplateFile template.json -WhatIf

Resource and property changes are indicated with this symbol:
  + Create

The deployment will update the following scope:

Scope: /subscriptions/.../resourceGroups/iac-what-if-poc-rg

  + Microsoft.Storage/storageAccounts/iacwhatifsa [2019-06-01]

      apiVersion:       "2019-06-01"
      id:               "/subscriptions/.../resourceGroups/iac-what-if-poc-rg/providers/Microsoft.Storage/storageAccounts/iacwhatifsa"
      kind:             "StorageV2"
      location:         "westeurope"
      name:             "iacwhatifsa"
      sku.name:         "Standard_LRS"
      tags.displayName: "iacwhatifsa"
      type:             "Microsoft.Storage/storageAccounts"

Resource changes: 1 to create.
```

As you can see, if we deploy this template now, there will be a `Create` operation and one resource will be created.

Let's deploy this template now, so we have this resource in resource group deployed.

```powershell
New-AzResourceGroupDeployment -ResourceGroupName iac-what-if-poc-rg -TemplateFile template.json
```

Now, let's change `sku` from `Standard_LRS` to `Standard_GRS` and check the execution plan.

```powershell
New-AzResourceGroupDeployment -ResourceGroupName iac-what-if-poc-rg -TemplateFile template.json -WhatIf

Resource and property changes are indicated with this symbol:
  ~ Modify

The deployment will update the following scope:

Scope: /subscriptions/.../resourceGroups/iac-what-if-poc-rg

  ~ Microsoft.Storage/storageAccounts/iacwhatifsa [2019-06-01]
    ~ sku.name: "Standard_LRS" => "Standard_GRS"

Resource changes: 1 to modify.
```

Now the plan shows that if you deploy, one resource will be modified.

## Practical usage

Now, how can we use this feature when it's in GA? Assuming that you practicing Infrastructure as code, your ARM templates are stored at git repository and all changes to your infrastructure go via similar flow.

```TXT
PR -> Code review -> PR Approve -> Merge to master -> Deploy
```

What we can do now is to run `-whatIf` command and check the deployment impact before we merge to master. It can be as simple as a "visual" check of what resources will be added/deleted/modified, or it could be some sort of automatic tests that verifies expected changes with actual changes provided by `-whatIf` command.

For example, if you use [github](https://github.com) as your source code and [Azure DevOps](https://azure.microsoft.com/nb-no/services/devops/) for your build and release pipelines, you can implement PR check that for each PR will trigger `Azure DevOps` build (or release) pipeline that will execute `-WhatIf` command with ARM templates from the branch. This way you can get an extra verification step that prevents you from deploying unverified changes of your infrastructure.

At one of the future posts I will setup CI/CD [Azure DevOps](https://azure.microsoft.com/nb-no/services/devops/) deployment pipeline for [Azure DNS Zones](https://docs.microsoft.com/en-us/azure/dns/). Stay tuned!

With that - thanks for reading!
