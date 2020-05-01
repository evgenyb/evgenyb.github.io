---
layout: post
title: "Use ARM template copy function to simplify NSG maintenance"
date: 2020-05-05
categories: [Azure DevOps]
---

Let's imagine that you implement your infrastructure as code with [ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) and you need to provision [Private Virtual Network](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) (further vnet) for different environments and you use different subnet configuration for different environments. Let's see how we can solve this.

## Define our infrastructure

Before we start, let's describe our environments specifications. We support 2 environments, `dev` and `prod` and here is `vnet` configurations:

```txt
VNet name: iac-dev-copy-poc-vnet
addressPrefix: 10.112.0.0/16

Subnets:
    name: aks-net
    addressPrefix: 10.112.0.0/20

    name: agw-net
    addressPrefix: 10.112.16.0/25
```

```txt
VNet name: iac-prod-copy-poc-vnet
addressPrefix: 10.111.0.0/16

Subnets:
    name: aks-net
    addressPrefix: 10.111.0.0/20

    name: agw-net
    addressPrefix: 10.111.16.0/25

    name: AzureBastionSubnet
    addressPrefix: 10.111.16.128/27
```

As you can see, in production there is one extra subnet to deploy [Azure Bastion](https://docs.microsoft.com/en-gb/azure/bastion/bastion-connect-vm-rdp).

## Create 2 resource groups

As always, we start by creating new resource group. Since we have 2 environments, we need to create 2 resource groups:

One for `dev`

```bash
az group create -n iac-dev-copy-poc-rg -l westeurope
```

and one for `prod`

```bash
az group create -n iac-prod-copy-poc-rg -l westeurope
```

## Some thoughts

How can we implement ARM template for our scenario? One obvious solution will be to implement 2 files, one for production and one for dev environments. We can use `-dev` and `-prod` suffixes to the template files. Let's do this.

### Create template file for `dev`

Let's create `template-dev.json` file containing ARM template for vnet resource.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
    },
    "functions": [
    ],
    "variables": {
    },
    "resources": [
        {
            "name": "iac-dev-copy-poc-vnet",
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2019-11-01",
            "location": "[resourceGroup().location]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.112.0.0/16"
                    ]
                },
                "subnets": [
                    {
                        "name": "aks-net",
                        "properties": {
                            "addressPrefix": "10.112.0.0/20"
                        }
                    },
                    {
                        "name": "agw-net",
                        "properties": {
                            "addressPrefix": "10.112.16.0/25"
                        }
                    }
                ]
            }
        }
    ],
    "outputs": {
    }
}
```

### Create template file for `prod`

Let's create `template-prod.json` file containing ARM template for vnet resource.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
    },
    "functions": [
    ],
    "variables": {
    },
    "resources": [
        {
            "name": "iac-prod-copy-poc-vnet",
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2019-11-01",
            "location": "[resourceGroup().location]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.111.0.0/16"
                    ]
                },
                "subnets": [
                    {
                        "name": "aks-net",
                        "properties": {
                            "addressPrefix": "10.111.0.0/20"
                        }
                    },
                    {
                        "name": "agw-net",
                        "properties": {
                            "addressPrefix": "10.111.16.0/25"
                        }
                    },
                    {
                        "name": "AzureBastionSubnet",
                        "properties": {
                            "addressPrefix": "10.111.16.128/25"
                        }
                    }
                ]
            }
        }
    ],
    "outputs": {
    }
}
```

If we go this route, then the need to use corresponding script for each environment when we deploy. So our deployment script could look like this one:

```bash
#!/usr/bin/env bash
# Usage:
#   ./deploy.sh dev
#   ./deploy.sh prod

environment=$1

az deployment group create -g iac-${environment}-copy-poc-rg --template-file template-${environment}.json
```

And then you can deploy your infrastructure by running this script

to dev

```bash
./deploy.sh dev
```

to prod

```bash
./deploy.sh prod
```

This is good and will work fine if you only have one resource in your template file, but what if your ARM template contains not only vnet resource, but some other resources? With "one template per environment" approach, we will duplicate all recourses per environment and that will be a maintenance nightmare.

What we want is one ARM template file and set of parameters file per environment. So how can we implement such a model?

As it turned out, we can by using 2 ARM templates futures:

* [use object as a parameter](https://docs.microsoft.com/en-us/azure/architecture/building-blocks/extending-templates/objects-as-parameters)
* [copy element](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/copy-resources)

According to the documentation
> By adding the copy element to the properties section of a resource in your template, you can dynamically set the number of items for a property during deployment. You also avoid having to repeat template syntax.

## Refactoring and conventions

So the idea is that we introduce subnets configuration as an array of objects, move it to the parameters file and use `copy` functionality of ARM template to implement vnet resource.

Let's also define some conventions:

* Template file will be called `template.json`
* Parameters files will be called `parameters-dev.json` and `parameters-prod.json` with the following parameters:

| Parameter name  | dev | prod |
|---|---|---|
| environment | dev | prod |
| location | westeurope | westeurope |
| vnetAddressPrefix | 10.112.0.0/16 | 10.111.0.0/16 |
| subnetsConfiguration | array of objects | array of objects |

* `vnet` name will be composed in the template based on `environment` parameter by using [concat()](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-array#concat) function.

Let's do this:

## Add parameters to the template

At the template file, define parameters at the parameters section

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "environment": {
           "type": "string"
        },
        "location": {
           "type": "string"
        },
        "vnetAddressPrefix": {
           "type": "string"
        },
        "subnetsConfiguration": {
            "type": "array"
        }
    },
    "functions": [
    ],
    "variables": {
    },
    "resources": [
        ...
    ]
}
```

## Create parameters file for dev environment

Create new file called `parameters-dev.json` and add parameters values for dev environment.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "environment": {
            "value": "dev"
        },
        "location": {
            "value": "westeurope"
        },
        "vnetAddressPrefix": {
            "value": "10.112.0.0/16"
        },
        "subnetsConfiguration": {
            "value": [
                {
                    "name": "aks-net",
                    "addressPrefix": "10.112.0.0/20"
                },
                {
                    "name": "agw-net",
                    "addressPrefix": "10.112.16.0/25"
                }
            ]
        }
    }
}
```

As you can see here, the `subnetsConfiguration` parameter is the array of objects with 2 fields: subnet `name` and subnet `addressPrefix`.

## Create parameters file for prod environment

Create new file called `parameters-prod.json` and add parameters values for prod environment.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "environment": {
            "value": "prod"
        },
        "location": {
            "value": "westeurope"
        },
        "vnetAddressPrefix": {
            "value": "10.111.0.0/16"
        },
        "subnetsConfiguration": {
            "value": [
                {
                    "name": "aks-net",
                    "addressPrefix": "10.111.0.0/20"
                },
                {
                    "name": "agw-net",
                    "addressPrefix": "10.111.16.0/25"
                }
            ]
        }
    }
}
```

## Refactor template file

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "environment": {
            "type": "string"
        },
        "location": {
            "type": "string"
        },
        "vnetAddressPrefix": {
            "type": "string"
        },
        "subnetsConfiguration": {
            "type": "array"
        }
    },
    "variables": {
        "vnetName": "[concat('iac-', parameters('environment'), '-copy-poc-vnet')]"
    },
    "resources": [
        {
            "name": "[variables('vnetName')]",
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2019-11-01",
            "location": "[parameters('location')]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "[parameters('vnetAddressPrefix')]"
                    ]
                },
                "copy": [
                    {
                        "name": "subnets",
                        "count": "[length(parameters('subnetsConfiguration'))]",
                        "input": {
                            "name": "[parameters('subnetsConfiguration')[copyIndex('subnets')].name]",
                            "properties": {
                                "addressPrefix": "[parameters('subnetsConfiguration')[copyIndex('subnets')].addressPrefix]"
                            }
                        }
                    }
                ]
            }
        }
    ]
}
```

Some comments about changes we did:

* We added variable `vnetName` and used `concat` function to compose the vnet name based on environment
* We replaced resource `location` with value from `location` parameter
* We replaced hard coded value of `addressPrefixes` property with value from `vnetAddressPrefix` parameter
* Finally, we used `copy` element of ARM template and implemented `subnets` property of `Microsoft.Network/virtualNetworks` resource from the `subnetsConfiguration` array`

In the context of our use-case, you can think of `copy` element as a `foreach` loop, where we iterate through each item of `subnetsConfiguration` array and create new array under `Microsoft.Network/virtualNetworks` resource properties called `subnets` where each new item of the new array will have the following structure

```json
{
    "name": "<>",
    "properties": {
        "addressPrefix": "<>"
    }
}
```

[copyIndex()](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-numeric#copyindex) function returns the current index in the `subnet` array.

It's not that easy to work with copy function, especially hard to debug, but actually there is one trick you can use if you stack with copy function for complex object. What you can do is you can use `output` section of ARM template to print the result of copy function without deploying template.
Let's create an empty arm template called `debug.json` with only one `subnetsConfiguration` parameter and implement the same copy function in output section.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "subnetsConfiguration": {
            "type": "array",
            "defaultValue": [
                {
                    "name": "aks-net",
                    "addressPrefix": "10.112.0.0/20"
                },
                {
                    "name": "agw-net",
                    "addressPrefix": "10.112.16.0/25"
                }
            ]
        }
    },
    "variables": {
    },
    "resources": [
    ],
    "outputs": {
        "copy-function-result": {
            "type": "array",
            "copy": {
                "count": "[length(parameters('subnetsConfiguration'))]",
                "input": {
                    "name": "[parameters('subnetsConfiguration')[copyIndex()].name]",
                    "properties": {
                        "addressPrefix": "[parameters('subnetsConfiguration')[copyIndex()].addressPrefix]"
                    }
                }
            }
        }
    }
}
```

Now let's deploy it

```bash
az deployment group create -g iac-dev-copy-poc-rg --template-file debug.json
```

and check the result's output section

```txt
{- Finished ..
...
  "properties": {
...
    "outputs": {
      "copy-function-result": {
        "type": "Array",
        "value": [
          {
            "name": "aks-net",
            "properties": {
              "addressPrefix": "10.112.0.0/20"
            }
          },
          {
            "name": "agw-net",
            "properties": {
              "addressPrefix": "10.112.16.0/25"
            }
          }
        ]
      }
    },
 ...
  "resourceGroup": "iac-dev-copy-poc-rg",
  "type": "Microsoft.Resources/deployments"
}
```

It contains the `copy` element's execution result. This way you can "debug" it while you implementing your ARm templates.

Now, we have one `template.json` file and 2 parameters files for each environments and we can change `deploy.sh` file.

```bash
#!/usr/bin/env bash
# Usage:
#   ./deploy.sh dev
#   ./deploy.sh prod

environment=$1

az deployment group create -g iac-${environment}-copy-poc-rg --template-file template-${environment}.json
```

as can deploy our infrastructure to dev

```bash
./deploy.sh dev
```

and to production

```bash
./deploy.sh prod
```

## Summary

`copy` function is quite powerful tool in ARM templates tool-belt. It's not very intuitive and sometimes quite hard to work with, but you can use trick to debug the result of `copy` element to `output` of ARM template deployment.

I used the same technique to implement other type of resources:

* [Network Security Group](https://docs.microsoft.com/en-us/azure/templates/microsoft.network/networksecuritygroups)  - here you can extract `securityRules` to the parameters files

* [Virtual Network Peering](https://docs.microsoft.com/en-us/azure/templates/microsoft.network/2019-11-01/virtualnetworks/virtualnetworkpeerings)

* [APIM custom domains](https://docs.microsoft.com/en-us/azure/templates/microsoft.apimanagement/2019-12-01/service) - you can implement environment specific custom domains with certificates via `hostnameConfigurations` section

But basically if you have a recitative properties, you can use `copy` element to implement it.

## Clean up

Don't forget to clean up you resources when you are done with exercise.

Remove `iac-dev-copy-poc-rg` resource group with all resources.

```bash
az group delete -n iac-dev-copy-poc-rg
```

Remove `iac-prod-copy-poc-rg` resource group with all resources.

```bash
az group delete -n iac-prod-copy-poc-rg
```

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!