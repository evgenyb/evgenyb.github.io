---
layout: post
title: "How to use APIM policy to lock down requests to specified Azure Front Door instance?"
date: 2020-07-28
categories: [Azure, APIM, API Management, APIM policy, Azure Front Door]
---

## APIM deployment models

If you want to use [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/#overview) in front of APIM instance, keep in mind that Front Door requires that [backends](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-backend-pool) are accessible from public internet. That means that APIM can only be added as Front Door backend when:

* APIM is not deployed into a virtual network
* APIM is deployed into virtual network with [external access type](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet)

![apim-internal](/images/2020-07-28-apim-external.png)

In both cases, API Management gateway is accessible from the public internet.

If you deploy APIM into virtual network with internal access type (this is when API Management gateway is accessible only from within the virtual network), then you need to [additionally provision Azure Application Gateway](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-integrate-internal-vnet-appgateway) in front of APIM and use it as a backend endpoint in Azure Front Door.

![apim-internal](/images/2020-07-28-apim-internal.png)

## APIM access restriction policies

For each requests sent to the backends, Front Door includes Front Door ID inside `X-Azure-FDID` header. If you want your APIM instance to only accept requests from Front Door, you can use the [check-header](https://docs.microsoft.com/en-us/azure/api-management/api-management-access-restriction-policies#CheckHTTPHeader) policy to enforce that a request has a `X-Azure-FDID` header and this header contains your Front Door ID. If the check fails, the policy terminates request processing and returns the HTTP status code and error message specified by the policy.

```xml
<check-header name="X-Azure-FDID" failed-check-httpcode="401" failed-check-error-message="Not authorized" ignore-case="false">
    <value>{{frontDoorId}}</value>
</check-header>
```

`frontDoorId` is a [APIM named value](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-properties) containing Front Door ID

## How to find Front Door ID?

### Portal

You can find Front Door ID value under the Overview section from Front Door portal page.

### az cli

If you have [front-door az cli extension](https://github.com/Azure/azure-cli-extensions/tree/master/src/front-door) installed.

```bash
az network front-door show -n iac-fd -g iac-base-rg --query frontdoorId
```

If you can't find `frontdoorId` field in the response, make sure that you use latest version of `front-door` extension.

Alternatively you can use this command.

```bash
az resource show  --api-version 2020-01-01 --resource-type Microsoft.Network/frontdoors --name iac-fd --resource-group iac-base-rg --query properties.frontdoorId
```

Note that you have to set `--api-version` flag to `2020-01-01` version or newer.

## Azure REST API

You can fetch the `frontdoorId` from Front Door’s management API. The easiest way to do this is going to https://docs.microsoft.com/en-us/rest/api/frontdoorservice/frontdoor/frontdoors/get and click “Try it”. Note that you have to use API version `2020-01-01` or newer for this.

## Network Security Group

If you deploy APIM into private virtual network (both for internal and external access types) and you only want to accept traffic from Front Door, you can use the service tag `AzureFrontDoor.Backend` in your [Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview) rules.

![apim-internal](/images/2020-07-28-nsg.png)

If you use `internal` access type, then you configure `AzureFrontDoor.Backend` rules at Network Security Group assigned to `agw-net` subnet (check image above). In addition, you can restrict that `api-net` subnet only accept traffic from `agw-net` subnet by configuring Network Security group assigned to `apim-net`.

## Useful links

* [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/#overview)
* [Backends and backend pools in Azure Front Door](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-backend-pool)
* [How do I lock down the access to my backend to only Azure Front Door?](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-faq#how-do-i-lock-down-the-access-to-my-backend-to-only-azure-front-door)
* [APIM named value](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-properties)
* [APIM access restriction policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-access-restriction-policies)
* [APIM check-header policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-access-restriction-policies#CheckHTTPHeader)
* [Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview)
* [Create an ingress controller to an internal virtual network in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/ingress-internal-ip)
* [How to use Azure API Management with virtual networks](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet)
* [Integrate API Management in an internal VNET with Application Gateway](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-integrate-internal-vnet-appgateway)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
