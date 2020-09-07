---
layout: post
title: "ARM template design choices and deployment time"
date: 2020-08-17
categories: [Azure, ARM templates, Infrastructure as Code, Immutable infrastructure]
---

When you use ARM templates to implement your infrastructure as code, there are multiple ways you can structure your ARM templates. When your infrastructure setup is small, it doesn't really matter, but when you work with complex infrastructure, with significant number of infrastructure components, the way how you structure your ARM templates directly affects how easy is to maintain them.

Let's have a look at the most common ways to structure ARM templates:

## #1 Single template

This is the "default" approach you will find it in documentation and most of the [Azure Resource Manager QuickStart Templates](https://github.com/Azure/azure-quickstart-templates). The biggest challenge with this approach is when there are a lot of resources in one ARM template, this json file becomes way too long and hard to read, navigate and maintain. If you use parameters to support multi-environment setup, these parameter files also become quite big and not so easy to work with.

Tools like [ARM Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools) help a lot and you get much better development experience and I highly recommend to use it.

In single template scenario you define the order for deploying resources by using [dependsOn element](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/define-resource-dependency?WT.mc_id=AZ-MVP-5003837#dependson) and Resource Manager orchestrates the deployment of the resources.

## #2 ARM template per resource

You don't have to define your entire infrastructure in a single template. Often, it makes sense to divide your infrastructure into a set of targeted templates.

If you use this approach, you would normally put template and parameters files into the folder representing one resource, or set of purpose-specific resources and it's your responsibility to orchestrate in which order "folders" should be deployed. To group several resources into one template, you use single template approach described in option #1.

Here is an example of such a folder structure with alphabetic ordering:

```txt
arm
    01-nsg  # contains Network Security Groups templates
        template.json   # single ARM template with several NSG resources
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

To deploy resources structured this way, you will either recursively iterate through folders structure and deploy each template with [az deployment group create](https://docs.microsoft.com/en-us/cli/azure/deployment/group?view=azure-cli-latest&WT.mc_id=AZ-MVP-5003837#az-deployment-group-create) command (or PowerShell equivalent), or create a master script that will deploy all resources one by one in the correct order.

My [How to live in harmony with ARM templates](https://borzenin.com/iac-ws1-labs/) workshop contains labs showing how to refactor from `Single ARM template` to set of `ARM template per resource`. Check out [lab03](https://github.com/evgenyb/iac-meetup/blob/master/workshops/01-how-to-live-in-harmony-with-ARM-templates/labs/lab-03/readme.md) and [lab04](https://github.com/evgenyb/iac-meetup/blob/master/workshops/01-how-to-live-in-harmony-with-ARM-templates/labs/lab-04/readme.md) to get more hands-on experience and here is the list of [all labs](https://github.com/evgenyb/iac-meetup/blob/master/workshops/01-how-to-live-in-harmony-with-ARM-templates/agenda.md) from the workshop.

Since your ARM templates are divided into smaller templates, it's easier to read and maintain both templates and parameters.  

## #3 Linked templates

Linked templates is a similar approach to `ARM template per resource`, but instead of deploying resources sequentially with master script, you create a master ARM template that links all the required templates.

When referencing a linked template, the value of uri must not be a local file or a file that is only available on your local network. You must provide a URI value that downloadable as http or https. Resource Manager must be able to access the templates and that makes this approach much more complex compare to other others.

When it comes to where to upload templates, one option is to place your linked template in a storage account, and use the URI with SAS token for them. If your templates are stable and don't contain any sensitive data, you can also store them in github repository.

For information about nested templates, see [Using linked templates with Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates?WT.mc_id=AZ-MVP-5003837).

The advantage of this method and method #1 compared to method #2 is that you don't have to worry about the complexities of ordering operations. Resource Manager orchestrates the deployment of interdependent resources so they're created in the correct order. When possible, Resource Manager deploys resources in parallel so your deployments finish faster than serial deployments. You deploy the template through one command, rather than through multiple imperative commands.

![template-processing](/images/2020-08-17-logo.png)

## Some thoughts about deployment time

Provisioning of Azure resources takes time and the more infrastructure components are in your environment, the more time it takes to provision new environment. If you use immutable infrastructure with blue-green provisioning model, then deployment time might not be so critical, but if your Disaster Recovery strategy is "redeploy on disaster", and during disaster, when you need to quickly provision new environment, then every minute counts.

If we look at the options from `time it takes to provision resource` perspective, then obviously options #1 and #3 perform best and #2 is the slowest one.

If you don't want to use Simple template or linked templates but still want to improve deployment time, you can introduce parallel deployment logic in your master script, but then you need to understand your resources dependencies and know what resources can be deployed in parallel. As you can imagine, that adds extra complexity, but technically it's possible.

## Let's get some numbers

Table below shows time it took to provision 2 Application Gateways (v2) and AKS cluster (1 node) sequentially:

Resource | Duration
---------|----------
AGW1     | 4 min 32 sec
AGW2     | 5 min 15 sec
AKS (1 node)     | 3 min 17 sec

In total, it took a little more than 13 min to provision all 3 resources.

To provision the same resources as a single ARM template it took 5 min 19 sec. More than two times faster!

## Useful links

* [What are ARM templates?](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview?WT.mc_id=AZ-MVP-5003837)
* [ARM template design](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview?WT.mc_id=AZ-MVP-5003837#template-design)
* [Define the order for deploying resources in ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/define-resource-dependency?WT.mc_id=AZ-MVP-5003837)
* [Using linked templates with Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates?WT.mc_id=AZ-MVP-5003837)
* [Azure Resource Manager QuickStart Templates](https://github.com/Azure/azure-quickstart-templates)
* [Azure Resource Manager Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools)
* [az deployment group create](https://docs.microsoft.com/en-us/cli/azure/deployment/group?view=azure-cli-latest&WT.mc_id=AZ-MVP-5003837#az-deployment-group-create)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading! :)
