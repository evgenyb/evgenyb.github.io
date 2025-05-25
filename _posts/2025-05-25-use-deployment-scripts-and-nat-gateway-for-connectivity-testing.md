---
layout: post
title: "Use Deployment Scripts and NAT Gateway to verify if your egress Public IP is whitelisted with you integration partners"
date: 2025-05-25
description: "This post describes how to use Deployment Scripts and NAT Gateway to verify if your egress Public IP is whitelisted with you integration partners."
image: "/images/2025-05-25-logo.png"
categories: ["Bicep", "NAT Gateway", "Infrastructure as Code", "Azure FIrewall", "Deployment Scripts"]
githubissuesid: 43
---

![logo](/images/2025-05-25-logo.png)

# POC description

My customer uses [Azure Secured Hub](https://learn.microsoft.com/en-us/azure/firewall-manager/secured-virtual-hub) with [Azure Firewall](https://learn.microsoft.com/en-us/azure/firewall/overview) where all outbound traffic is routed through Azure Firewall egress IP. 
Customer decided to switch to [Bring Your Own IP version of Azure Firewall](https://learn.microsoft.com/en-us/azure/firewall/secured-hub-customer-public-ip) (at the time of writing is still in Public Preview). Customer services communicate with a lot of partners APIs and some of them require whitelisting of Azure Firewall IP. 
Following [recommendations](https://learn.microsoft.com/en-us/azure/firewall/firewall-known-issues), to avoid `SNAT port exhaustion`, Azure Firewall has to be configured with a minimum of 5 public egress IP addresses. This means that before customer can switch to new Azure Firewall, new set of IP addresses `must be` whitelisted with all partners. [Disaster Recovery](https://azure.microsoft.com/en-us/resources/cloud-computing-dictionary/what-is-disaster-recovery) (aka DR) plan is also affected, as customer needs to request partners to whitelist new set of IP addresses for DR Azure Firewall as well. 

# The challenge

There will be 16 new IP addresses that need to be whitelisted by partners and we need a way to test that `each` partner actually whitelisted `all` new IP addresses. So, we need to setup a sort of lab environment where we can:
- configure outbound traffic to be routed through one specified IP address
- execute TCP test towards each partner IP / FQDN at the specified port(s)
- run these tests automatically every day/week/month and generate report showing whitelisting progress

# The solution

After some whiteboarding and head scratching, we came up with the following solution for this test lab environment:

 - use [Public IP Prefix](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-address-prefix) with `/29` subnet mask to allocate 8 IP addresses for active environment and another `/29` for DR environment. We use Prefix resource because we will know all new IP values from the IP range in advance
 - provision all Public IPs from the Prefixes during test session
 - use [Azure Deployment Scrips integrated with Private VNet](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-script-vnet) to execute connectivity tests
 - use [NAT Gateway](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-overview) and configure Deployment Scripts subnet to route outbound traffic through one IP address
 - use PowerShell [Test-Connection](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/test-connection?view=powershell-7.5) command to test connectivity
 
Deployment Scripts will be executed from within VNet, and VNet is configured to route outbound traffic via NAT Gateway. Since we can programmatically change NAT GW IP address, we can execute connectivity tests towards partners for each of the available Public IP addresses.

Here is the pseudo-code of this POC solution:

- provision lab environment 
- for each IP from available IP list
 - configure NAT Gateway to use specified IP address
 - deploy Deployment Script and execute connectivity tests towards partners
 - collect results
 - repeat 
- generate report 
- decommission lab environment

# The implementation

The POC implementation is located in [this repo](https://github.com/iac-oslo/natgw-poc) inside the `iac` folder. The infra code is orchestrated by `main.bicep` script and all infrastructure resources are collected inside `modules/infra.bicep` file. 

Very simple version of connectivity tests towards partner IP / FQDN is located in `testPartners.ps1` file. The list of partners is implemented as an array of strings. 

```powershell
...
 $partners = @(
    'Foo,ifconfig.me,443',
    'Bar,ifconfig.me,80',
    'FooBar,ifconfig.me,22'
)
...
```
The `Deployment Scripts` code is stored in `test.bicep` file. It reads the content of `testPartners.ps1` file at deployment time using [loadTextContent](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions-files#loadtextcontent) function and sends it to the [deploymentScripts](https://learn.microsoft.com/en-us/azure/templates/microsoft.resources/deploymentscripts?pivots=deployment-language-bicep) resource. 

```powershell
...
scriptContent: loadTextContent('testPartners.ps1')
...
```

We used `az cli` to change the NAT Gateway Public IP address.

```powershell
...
az network nat gateway update -g $resourceGroupName -n "natgw-$prefix" --public-ip-addresses "pip-$prefix-$i" --output none
...
```

The script execution results exposed via Bicep `output` property.

```powershell
...
output results array = dsTest.properties.outputs.results
```

and then queried and outputted to the console from the deployment command:

```powershell
...
az deployment group ...  --query properties.outputs.results.value
```

Here is the result of executing this script at my environment `deploy-and-test.ps1`... 

```powershell
❯❯ iac git:(main) 23:33 .\deploy-and-test.ps1
Deploying testlab infra...
Test Partners...
Updating NAT Gateway with new pip-natgw-poc-1 IP address...
Deploy and run test script...
[
  "Outbound IP - 20.100.26.81",
  "Testing Foo at ifconfig.me:443 - True",
  "Testing Bar at ifconfig.me:80 - True",
  "Testing FooBar at ifconfig.me:22 - False"
]
Updating NAT Gateway with new pip-natgw-poc-2 IP address...
Deploy and run test script...
[
  "Outbound IP - 20.100.26.83",
  "Testing Foo at ifconfig.me:443 - True",
  "Testing Bar at ifconfig.me:80 - True",
  "Testing FooBar at ifconfig.me:22 - False"
]
Updating NAT Gateway with new pip-natgw-poc-3 IP address...
Deploy and run test script...
[
  "Outbound IP - 20.100.26.82",
  "Testing Foo at ifconfig.me:443 - True",
  "Testing Bar at ifconfig.me:80 - True",
  "Testing FooBar at ifconfig.me:22 - False"
]
Updating NAT Gateway with new pip-natgw-poc-4 IP address...
Deploy and run test script...
[
  "Outbound IP - 20.100.26.80",
  "Testing Foo at ifconfig.me:443 - True",
  "Testing Bar at ifconfig.me:80 - True",
  "Testing FooBar at ifconfig.me:22 - False"
]
Deployment time:  00:08:53.6863343
```

This output can be used to generate a report showing the progress of whitelisting, but this is outside of the scope of this POC.

# Cleaning up

If you deployed this POC into your environment, don't forget to clean up the resources when you finished.

```powershell
az group delete --resource-group rg-natgw-poc --yes
```



With that - thanks for reading!
