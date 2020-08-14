---
layout: post
title: "ARM template design choices and deployment time"
date: 2020-08-18
categories: [Azure, ARM templates, Infrastructure as Code]
---

When you use ARM templates to implement your infrastructure as code, there are multiple ways you can structure your ARM templates. When your infrastructure setup is small it doesn't really matter, but if you are working with complex infrastructure, containing significant number of different infrastructure components, the way how you maintain your ARM template becomes much more important.

Let's look at the most common template structuring techniques:

## #1 Single template

This is the "default" approach you will find in all documentation and 101 examples. The biggest challenge with this approach is that ARM template json file becomes way too big and hard to maintain. With [Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools) plugin, you will get much better development experience. If you use parameters (for multi environment setup) to define components' configuration, these parameter files become also quite big and not so easy to maintain.

In single template scenario you define the order for deploying resources by using [dependsOn element](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/define-resource-dependency#dependson).

## #2 ARM template per resource

You don't have to define your entire infrastructure in a single template. Often, it makes sense to divide your infrastructure into a set of targeted templates.

If you go that path, you would normally put template and parameters files into the folder representing one resource, or set of purpose-specific resources. In that scenario, it's your responsibility to orchestrate in which order resources will be provisioned.

Here is an example of such a folder structure with alphabetic ordering:

```txt
arm
    01-nsg  # contains Network Security Groups templates
        template.json  
        parameters.json
        deploy.sh
    02-vnet # contains Virtual Private Network template
        template.json
        parameters.json
        deploy.sh
    03-agw   # contains Application Gateway template
        template.json
        parameters.json
        deploy.sh
    04-aks   # contains Azure Kubernetes Service template
        template.json
        parameters.json
        deploy.sh
    05-apim  # contains API Management template
        template.json
        parameters.json
        deploy.sh
    deploy.sh
```

To provision this infrastructure, you will either resourcefully iterate through folders structure and provision each resource with `az deploy resource` command (or PowerShell equivalent), or create a master script that will provision all resources one by one.

My [How to live in harmony with ARM templates](https://borzenin.com/iac-ws1-labs/) workshop contains labs showing how you can refactor  `Single ARM template` to set of `ARM template per resource`. Check out [lab03](https://github.com/evgenyb/iac-meetup/blob/master/workshops/01-how-to-live-in-harmony-with-ARM-templates/labs/lab-03/readme.md) and [lab04](https://github.com/evgenyb/iac-meetup/blob/master/workshops/01-how-to-live-in-harmony-with-ARM-templates/labs/lab-04/readme.md). Here is the list of [all labs](https://github.com/evgenyb/iac-meetup/blob/master/workshops/01-how-to-live-in-harmony-with-ARM-templates/agenda.md).

## #3 Linked templates

Linked templates is a similar approach as `ARM template per resource`, but instead of deploying resources sequentially with master bash or  PowerShell script, you create a master ARM template that links all the required templates.

When referencing a linked template, the value of uri must not be a local file or a file that is only available on your local network. You must provide a URI value that downloadable as http or https. Resource Manager must be able to access the templates and that makes this approach much more complex compare to other others.

When it comes to where to upload templates, one option is to place your linked template in a storage account, and use the URI with SAS token for that item.  

For information about nested templates, see [Using linked templates with Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates).

The advantage of this method (compared to `ARM template per resource` method) is that you don't have to worry about the complexities of ordering operations. Resource Manager orchestrates the deployment of interdependent resources so they're created in the correct order. When possible, Resource Manager deploys resources in parallel so your deployments finish faster than serial deployments. You deploy the template through one command, rather than through multiple imperative commands.

![emplate-processing](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/media/overview/template-processing.png)


## Some thoughts about performance

Provisioning of Azure resources takes time. The more infrastructure components are in your environment, the more time it will take to provision new environment. If you employ blue-green provisioning mordel 

## Let's get some numbers

## Useful links

* [What are ARM templates?](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview)
* [ARM template design](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview#template-design)
* [Define the order for deploying resources in ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/define-resource-dependency)
* [Using linked templates with Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
