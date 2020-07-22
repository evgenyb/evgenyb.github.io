---
layout: post
title: "How to use APIM policy for canary testing"
date: 2020-07-25
categories: [Azure, APIM, API Management, Canary, APIM policy]
---

![logo](/images/2020-07-25-logo.png)

One of the benefits using immutable infrastructure is that it allows you to do a canary testing of your infrastructure. That is - you provision new version of your infrastructure components, deploys your services and then route small percentage of the traffic towards new infrastructure.

To implement canary traffic orchestration, such products as [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) with [weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-routing-methods#weighted) and / or [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/#overview) with [weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-routing-methods#weighted) are normally used.
But what if you use [API Management](https://azure.microsoft.com/en-us/services/api-management/) in front of your services? How can we distribute traffic between services running at the current and vNext versions of your infrastructure?

## Use-case description

Here is how our infrastructure looks like.

![logo](/images/2020-07-25-use-case.png)

* We use Azure Front Door in front of APIM
* APIM instance is deployed to private VNet with [external access type](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet). That means that APIM gateway is accessible from the public internet and gateway can access resources within the virtual network
* APIM [Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview) is configured to only accept traffic from Azure Front Door
* AKS is deployed to private VNet peered with APIM Vnet and NSG only accepting traffic from `apim-net` subnet
* We use [nginx private ingress controller](https://docs.microsoft.com/en-us/azure/aks/ingress-internal-ip)
* We use `blue` / `green` convention to mark infrastructure "version"
* Normally, there is only one version of AKS cluster is active, but when we introduce some infrastructure changes (for example, we want to upgrade AKS cluster to the newer version, or change AKS VM size or do some other AKS related changes), we have 2 active AKS clusters

## APIM policy configuration

 [APIM policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-policies) - a collection of Statements that are executed sequentially on the request or response of an API. Popular Statements include format conversion from XML to JSON and call rate limiting to restrict the amount of incoming calls. Here is the full policy [reference index](https://docs.microsoft.com/en-us/azure/api-management/api-management-policies).

To implement canary flow orchestration between 2 AKS clusters at APIM, we can use [control flow](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#choose) and [set backend service](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetBackendService) policy.

Here is the example of API level policy for API called `api-b`.

```xml
<policies>
    <inbound>
        <base />
        <choose>
            <when condition="@(new Random().Next(100) < {{canaryPercent}})">
                <set-backend-service base-url="http://{{aksHostCanary}}/api-b/" />
            </when>
            <otherwise>
                <set-backend-service base-url="http://{{aksHost}}/api-b/" />
            </otherwise>
        </choose>
    </inbound>
...
</policies>
```

* `canaryPercent` is [APIM named value](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-properties) specifying the percentage of the requests we want to send to the new AKS cluster
* `aksHost` contains AKS ingress controller private IP address (or domain name pointing towards this IP) of current AKS cluster (in our use-case, `aks-dev-green`)
* `aksHostCanary` contains AKS ingress controller private IP address (or domain name pointing towards this IP) of new version of AKS cluster (in our use-case, `aks-dev-blue`)

That works fine for one or two APIs, but if there are hundreds of APIs, that might be a bit too much XML "noise" in every API policy. To simplify it, we can define the `aksUrl` based on our  "canary" logic at Global level policy and use this variable at `set-backend-service` policy at API level policy.

### Global level policy

```xml
<policies>
    <inbound>
        <choose>
            <when condition="@(new Random().Next(100) < {{canaryPercent}})">
                <set-variable name="aksUrl" value="http://{{aksHostCanary}}" />
            </when>
            <otherwise>
                <set-variable name="aksUrl" value="http://{{aksHost}}" />
            </otherwise>
        </choose>
    </inbound>
...
</policies>
```

### api-b API level policy

```xml
<policies>
    <inbound>
        <base />
        <set-backend-service base-url="@((string)context.Variables["aksUrl"] + "/api-b/")" />
    </inbound>
...
</policies>
```

THis way we keep the canary "business" logic in one place (global level policy) and if change this logic, there will be done at one place only.

## Switching scenario

With this setup in place, here is how we can introduce new 

## Wrapping it up

## Useful links

* [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview)
* [Azure Traffic Manager weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-routing-methods#weighted)
* [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/#overview)
* [Azure Front Door weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-routing-methods#weighted) 
* [APIM control flow policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#choose)
* [APIM set backend service](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetBackendService)
* [APIM set backend service policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetBackendService)
* [APIM named value](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-properties)
* [Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview)
* [Create an ingress controller to an internal virtual network in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/ingress-internal-ip)
* [How to use Azure API Management with virtual networks](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
