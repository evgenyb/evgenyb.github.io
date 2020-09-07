---
layout: post
title: "Import existing Azure DNS Zone into Pulumi"
date: 2020-09-08
categories: [Azure, Pulumi, Infrastructure as Code, Azure DNS Zone]
---

I am currently working with the materials for my upcoming [Workshop #3: Implement immutable infrastructure on Azure with Pulumi](https://www.meetup.com/Infrastructure-As-Code-User-Group-Oslo/events/272952783/) and spend a lot of time working with [Pulumi on Azure](https://www.pulumi.com/docs/get-started/azure/). To have a realistic POC to work with, I decided to implement CI/CD pipeline for my [Azure DNS Zone](https://docs.microsoft.com/en-us/azure/dns/dns-zones-records?WT.mc_id=AZ-MVP-5003837) that handles DNS records for my `iac-labs.com` domain.

## Importing Infrastructure

Since this Zone was already provisioned manually before and already contains several records, I need to adopt existing resources under management so they can be managed by Pulumi.

To adopt existing resources so that Pulumi is able to manage subsequent updates to them, Pulumi offers the [import](https://www.pulumi.com/docs/intro/concepts/programming-model/#import) resource option. This option request that a resource defined in your Pulumi program adopts an existing resource in the cloud provider instead of creating a new one as would normally occur.

In my case, to import DNS Zone, I need to import the following resources:

* resource group where DNS Zone is deployed
* DNS Zone instance
* all records (A, CName, NS, TXT, etc...)

For each of the resources you want to import, you need to find it's Azure resource ID. Normally, you can find this information at the `Properties` page at the Azure portal, but some of the resources, like DNS Zone records, don't show this information at the portal, but you can get resource ID with [az cli](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest?WT.mc_id=AZ-MVP-5003837).

This script gives me my Resource Group ID

```bash
az group show -g iac-domains-and-certificates-rg --query id
```

This script gives me my DNS Zone ID

```bash
az network dns zone show -n iac-labs.com -g iac-domains-and-certificates-rg --query id
```

and this script gives me list of all record sets 

```bash
az network dns record-set list --zone-name iac-labs.com -g iac-domains-and-certificates-rg
```

With ID in place, what we need to do now is implement normal Pulumi stack with resource group, DNS Zone and records, but in addition specify `ImportId` for each of the resources you want to adapt.

Here is the code that imports resource group, Azure DNS Zone instance and TXT, NS and CName record sets.

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

## Useful links

* [Pulumi](https://www.pulumi.com/)
* [Pulumi on Azure](https://www.pulumi.com/docs/get-started/azure/)
* [Importing Infrastructure](https://www.pulumi.com/docs/guides/adopting/import/)
* [Pulumi import](https://www.pulumi.com/docs/intro/concepts/programming-model/#import)
* [Overview of DNS zones and records](https://docs.microsoft.com/en-us/azure/dns/dns-zones-records?WT.mc_id=AZ-MVP-5003837)
* [Manage DNS records and recordsets in Azure DNS using the Azure CLI](https://docs.microsoft.com/en-us/azure/dns/dns-operations-recordsets-cli?WT.mc_id=AZ-MVP-5003837)
* [Azure Command-Line Interface (CLI) documentation](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest?WT.mc_id=AZ-MVP-5003837)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading! :)
