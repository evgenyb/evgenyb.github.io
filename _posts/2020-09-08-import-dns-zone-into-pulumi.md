---
layout: post
title: "Import existing Azure DNS Zone into Pulumi"
date: 2020-09-08
categories: [Azure, Pulumi, Infrastructure as Code, Azure DNS Zone]
---

I am currently working with the materials for my upcoming [Workshop #3: Implement immutable infrastructure on Azure with Pulumi](https://www.meetup.com/Infrastructure-As-Code-User-Group-Oslo/events/272952783/) and spending a lot of time working with [Pulumi on Azure](https://www.pulumi.com/docs/get-started/azure/). To have a realistic POC to work with, I decided to implement CI/CD pipeline for my [Azure DNS Zone](https://docs.microsoft.com/en-us/azure/dns/dns-zones-records?WT.mc_id=AZ-MVP-5003837) that handles DNS records for my `iac-labs.com` domain.

## Importing Infrastructure

Since this Zone was already provisioned manually long before and already contains several records, I need to import existing resources so they can be managed by Pulumi.

To adopt existing resources so that Pulumi is able to manage subsequent updates to them, Pulumi offers the [import](https://www.pulumi.com/docs/intro/concepts/programming-model/#import) resource option. This option request that a resource defined in your Pulumi program adopts an existing resource in Azure instead of creating a new one.

In my case, to import DNS Zone, I need to import the following resources:

* resource group where DNS Zone is deployed
* DNS Zone instance
* all DNS record sets (A, CName, NS, TXT, etc...)

For each of the resources you want to import, you need to find it's Azure resource ID. Normally, you can find this information at the `Properties` page at the Azure portal, but some of the resources, like DNS record sets, don't show this information at the portal, but you can always get resource ID with [az cli](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest?WT.mc_id=AZ-MVP-5003837).

The following script gives me my Resource Group ID

```bash
az group show -g iac-domains-and-certificates-rg --query id
```

and this script gives me my DNS Zone ID

```bash
az network dns zone show -n iac-labs.com -g iac-domains-and-certificates-rg --query id
```

and finally, this script gives me the list of all record sets under my DNS Zone

```bash
az network dns record-set list --zone-name iac-labs.com -g iac-domains-and-certificates-rg
```

With ID in place, what I need to do now is to implement the regular Pulumi stack including resource group, DNS Zone and record sets, but in addition to normal resource specification, specify `ImportId` for each of the resources you want to adapt.

Here is the code that imports resource group, Azure DNS Zone instance and TXT, NS and CName record sets. I am .net guy, so I will use C#, but if you use other [languages supported by Pulumi](https://www.pulumi.com/docs/intro/languages/), I hope it will be easy for you to follow...

```c#
// Create an Azure Resource Group
var resourceGroup = new ResourceGroup("rg", new ResourceGroupArgs
{
    Name = "iac-domains-and-certificates-rg"
}, new CustomResourceOptions {
    // Import existing resource
    ImportId = "/subscriptions/SUBSCRIPTIONID/resourceGroups/iac-domains-and-certificates-rg"
});

var dnsZone = new Zone("iac-com", new ZoneArgs
{
    Name = "iac-labs.com",
    ResourceGroupName = resourceGroup.Name,
}, new CustomResourceOptions {
    // Import existing resource
    ImportId = "/subscriptions/SUBSCRIPTIONID/resourceGroups/iac-domains-and-certificates-rg/providers/Microsoft.Network/dnszones/iac-labs.com"
});

var nsRecord = new NsRecord("ns", new NsRecordArgs
{
    Name = "@",
    ZoneName = dnsZone.Name,
    ResourceGroupName = resourceGroup.Name,
    Ttl = 172800,
    Records =
    {
        "ns1-08.azure-dns.com.",
        "ns2-08.azure-dns.net.",
        "ns3-08.azure-dns.org.",
        "ns4-08.azure-dns.info.",
    }
}, new CustomResourceOptions {
    ImportId = "/subscriptions/SUBSCRIPTIONID/resourceGroups/iac-domains-and-certificates-rg/providers/Microsoft.Network/dnszones/iac-labs.com/NS/@"
});

new CNameRecord("iac-lab-portal", new CNameRecordArgs
{
    Name = "iac-lab-portal",
    ZoneName = dnsZone.Name,
    ResourceGroupName = resourceGroup.Name,
    Ttl = 300,
    Record = "iac-lab-green-api-agw.westeurope.cloudapp.azure.com"
}, new CustomResourceOptions {
    ImportId = "/subscriptions/SUBSCRIPTIONID/resourceGroups/iac-domains-and-certificates-rg/providers/Microsoft.Network/dnszones/iac-labs.com/CNAME/iac-lab-portal"
});

```

`pulumi preview` gives me the following information

```bash
$ pulumi preview
Previewing update (prod)
...

     Type                         Name            Plan       
 +   pulumi:pulumi:Stack          dns-zone-prod   create     
 =   ├─ azure:core:ResourceGroup  rg              import     
 =   ├─ azure:dns:Zone            iac-com         import     
 =   ├─ azure:dns:CNameRecord     iac-lab-portal  import     
 =   └─ azure:dns:NsRecord        ns              import     
 
Resources:
    + 1 to create
    = 4 to import
    5 changes
```

That means that if I run `pulumi up`, a new stack will be cerated and 5 resources will be imported.

After I successfully import all resources, I can remove `new CustomResourceOptions { ImportId = "" }` code blocks to make my code clean and nice.

## Azure DevOps CI/CD pipeline

I what to use Azure DevOps pipelines to deploy new DNS records with the following flow:

* `master` branch represents what is deployed to DNS Zone
* If we need to create new, update or delete existing record set, developer adds change into Pulumi code and creates new pull request
* Pull requests triggers pipeline that runs `pulumi preview` command
* The person that does PR review needs to check the result of the `preview` command and only accept PR if it does what it supposed to do
* When PR is merged, it triggers pipeline that runs `pulumi up --yes` command that deploys changes to DNS Zone.

There is [Pulumi Task Extension](https://marketplace.visualstudio.com/items?itemName=pulumi.build-and-release-task) available at Visual Studio Marketplace that lets you easily use Pulumi in your CI/CD pipelines. It can be used both with classic Azure DevOps releases and YAML based pipelines.

## Useful links

* [Pulumi](https://www.pulumi.com/)
* [Languages supported by Pulumi](https://www.pulumi.com/docs/intro/languages/)
* [Pulumi on Azure](https://www.pulumi.com/docs/get-started/azure/)
* [Importing Infrastructure](https://www.pulumi.com/docs/guides/adopting/import/)
* [Pulumi import](https://www.pulumi.com/docs/intro/concepts/programming-model/#import)
* [Overview of DNS zones and records](https://docs.microsoft.com/en-us/azure/dns/dns-zones-records?WT.mc_id=AZ-MVP-5003837)
* [Manage DNS records and recordsets in Azure DNS using the Azure CLI](https://docs.microsoft.com/en-us/azure/dns/dns-operations-recordsets-cli?WT.mc_id=AZ-MVP-5003837)
* [Azure Command-Line Interface (CLI) documentation](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest?WT.mc_id=AZ-MVP-5003837)
* [Pulumi CD with Azure DevOps](https://www.pulumi.com/docs/guides/continuous-delivery/azure-devops/)
* [Pulumi Azure task extension for Azure Pipelines](https://marketplace.visualstudio.com/items?itemName=pulumi.build-and-release-task)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading! :)
