---
layout: post
title: "How to use Azure API Management policy for canary testing"
date: 2020-07-23
categories: [Azure, APIM, API Management, Canary testing, APIM policy, Azure Front Door, Azure Traffic Manager, Immutable infrastructure]
---

One of the benefits of using immutable infrastructure is that it allows you to do a canary testing of your infrastructure. That is - you provision new version of your infrastructure components, deploy your services and then route small percentage of the traffic towards new infrastructure, monitor how apps work under the new infra and eventually switch all traffic to new infrastructure.  

To implement canary traffic orchestration, such products as [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) with [weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-routing-methods#weighted) or [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/#overview) with [weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-routing-methods#weighted) are normally used.
But what if you use [Azure API Management](https://azure.microsoft.com/en-us/services/api-management/) in front of your services? How can you distribute traffic between services running at the current and vNext versions of your infrastructure?

## Use-case description

Let's consider the following hypothetical infrastructure setup.

![use-case](/images/2020-07-25-use-case.png)

* We use Azure Front Door in front of APIM
* APIM instance is deployed to private VNet with [external access type](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet). That means that APIM gateway is accessible from the public internet and gateway can access resources within the virtual network
* We don't want direct access to APIM from internet, therefore APIM [Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview) is configured to only accept traffic from Azure Front Door
* AKS is deployed to private VNet peered with APIM Vnet
* `aks-net` NSG configured to only accept traffic from `apim-net` subnet
* We use AKS [nginx private ingress controller](https://docs.microsoft.com/en-us/azure/aks/ingress-internal-ip)
* We use `blue` / `green` convention to mark our infrastructure versions
* Normally, there is only one version of AKS cluster is active, but when we introduce some infrastructure changes (for example, we want to upgrade AKS cluster to the newer version, or change AKS VM size or do some other AKS related changes), we have 2 active AKS clusters

## APIM policy configuration

 [APIM policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-policies) - a collection of Statements that are executed sequentially on the request or response of an API. Popular Statements include format conversion from XML to JSON and call rate limiting to restrict the amount of incoming calls. Here is the full policy [reference index](https://docs.microsoft.com/en-us/azure/api-management/api-management-policies).

To implement canary flow orchestration between 2 AKS clusters, we can use [control flow](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#choose) and [set backend service](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetBackendService) policies.

Here is the example of API level policy for API named `api-b`. Assuming that `api-b` backend service is accessible via the following AKS endpoints:

* http://10.2.15.10/api-b/ at `aks-dev-blue`
* http://10.3.15.10/api-b/ at `aks-dev-green`

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

We use [APIM named values](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-properties) to store our configurable variables:

* `canaryPercent` contains the percentage (value from 0 to 100) of the requests we want to send to canary cluster
* `aksHost` contains AKS ingress controller private IP address (`10.2.15.10`) of current AKS cluster (in our use-case, `aks-dev-green`)
* `aksHostCanary` contains AKS ingress controller private IP address (`10.3.15.10`) of next version of AKS cluster (in our use-case, `aks-dev-blue`)

This approach works fine for one or two APIs, but if there are hundreds of APIs under APIM, that might be a bit too much XML "noise" in every API policy and duplication of canary orchestration logic all over the place. To solve this, we can move canary logic to Global level policy, put the result of canary logic into the variable `aksUrl` using [set-variable](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#set-variable) policy and then use this variable at `set-backend-service` policy at API level policy.

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
        <set-backend-service base-url="@(context.Variables.GetValueOrDefault<string>("aksUrl") + "/api-b/")" />
    </inbound>
...
</policies>
```

This way, we keep the canary "business" logic in one place (Global level policy) and if we need to change this logic, the change will be done at one place only.

## Switching scenario

With this setup in place, here is the typical switch scenario:

1. The current environment is `blue`. APIM named values state:

* `aksHost` = 10.2.15.10
* `canaryPercent` = 0

2. We provision `green` environment and want to send 10% of the traffic to `green`. APIM named values state:

* `aksHost` = 10.2.15.10
* `aksCanaryHost` = 10.3.15.10
* `canaryPercent` = 10

3. We want to increase canary traffic to 50%. APIM named values state:

* `aksHost` = 10.2.15.10
* `aksCanaryHost` = 10.3.15.10
* `canaryPercent` = 50

3. Everything looks good and we want 100% of the traffic go to `green`. APIM named values state:

* `aksHost` = 10.3.15.10
* `canaryPercent` = 0

At that moment `green` is our active environment and we can decommission `blue`.

## Useful links

* [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview)
* [Azure Traffic Manager weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-routing-methods#weighted)
* [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/#overview)
* [Azure Front Door weighted traffic-routing method](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-routing-methods#weighted)
* [APIM control flow policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#choose)
* [APIM set variable policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#set-variable)
* [APIM set backend service policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetBackendService)
* [APIM named value](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-properties)
* [Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview)
* [Create an ingress controller to an internal virtual network in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/ingress-internal-ip)
* [How to use Azure API Management with virtual networks](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
