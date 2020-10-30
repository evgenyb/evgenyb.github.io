---
layout: post
title: "How to do blue-green testing with Azure Front Door and API Management"
date: 2020-10-29
categories: [APIM, API Management, Canary testing, APIM policy, Azure Front Door, IaC, Infrastructure As Code]
---

![logo](/images/2020-10-29-logo.png)

If you use Azure API Management and want to adapt blue-green deployment or provisioning model, APIM has functionality that allows you to implement a [canary traffic orchestration](https://borzenin.com/apim-canary-policy/) at the policy level.
Here is a typical scenario for how it works:

* you have an active version of your infrastructure provisioned and deployed to a slot that we will call the `blue` slot
* you introduce a new version of your infrastructure and deploy it into a new slot - the `green` slot
* you configure APIM to send some small percentage (let's say 10%) of the traffic to the `green` slot
* you monitor logs and if all looks good, you increase the percentage to 50%
* eventually you switch all 100% of the traffic to the `green` slot and decommission the `blue` one

That works fine, but sometimes you want to test your new version (in the example above that's the `green` slot) before you open traffic even for canary testing. How do you do this?

The easiest solution is to enrich requests by adding extra header with information about which `slot` we want to redirect traffic to. Let's call this header `Redirect-To` with supported values `blue` or `green`. With this header added to the request, you can implement APIM policy and route the requests to the backend specified at the `Redirect-To` header.

If you are lucky and have control of your system consumers, you can enrich requests at the client side, but more often than not, you can't do this and then the question is what options are available?

## Use Azure Front Door

The following Azure Front Door concepts will help us to solve our task:

### Additional frontends

When you create Azure Front Door, you will have at least one frontend with hostname matching your Front Door instance name + `.azurefd.net`. Normally, you will add a custom domain with your domain name, for instance `api.foo-bar.org` which is configured as a CNAME record, pointing to your original FD host name.  
When I want to test the new inactive (`green`) slot, I can add an extra custom domain endpoint, for example `api29cc67d2.foo-bar.org`, where `29cc67d2` is just random id, but you can use some more meaningful domain names like `api-inactive.foo-bar.org`.

### Rules Engine

Azure Front Door has a concept of [Rules Engine](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine?WT.mc_id=AZ-MVP-5003837) that allows you to customize how HTTP requests are handled and provides a more controlled behavior to back-ends. It supports several [actions](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837) and the one that interests us is [Modify request header](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837#modify-request-header). This action allows you to modify headers that are present in requests sent to your origin.

Check this [tutorial](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-tutorial-rules-engine?WT.mc_id=AZ-MVP-5003837) and learn how to configure your Rules Engine.

With Rules engine I can configure the following rule (pseudo code):

```txt
If requestUrl contains `29cc67d2` then add new request header Redirect-To and set its value to green
```

Here is how it looks when you edit it at the Portal

![AFD-rules-engine](/images/2020-10-29-AFD-rules-engine.png)

### Routing rules

We also need to add new routing rule `api-inactive` that routes all traffic from `api29cc67d2.foo-bar.org` frontend to `apim-backend` Backend pool. This rule has to be configured to use `BlueGreenRules` Rules Engine.

Here is how it looks at the Portal

![routing-rule-blue](/images/2020-10-29-routing-rule-blue.png)

### APIM policies

At the API Management side we need to implement a `choose` policy that checks if requests contain `Redirect-To` header, and if so, extracts header value and uses `set-backend-service` policy to redirect the  incoming request to a corresponding beckend (either `blue` or `green`).  

### Putting it all together

#### Traffic to inactive slot

With this setup in place, when I call "inactive" frontend

```bash
curl --get https://api29cc67d2.foo-bar.org/foo
```

the following set of actions will take place:

![flow](/images/2020-10-29-inactive-flow.png)

* Azure Front Door receives traffic from the `api29cc67d2.foo-bar.org` frontend
* AFD uses `api-inactive` Routing rule
* AFD identifies that `api-inactive` Routing rule uses `BlueGreenRules` Rules engine
* The conditions of `BlueGreenRules` Engine rule are met and AFD will add `Redirect-To` header with value `green` to the request
* Request is sent to the APIM backend
* APIM policy identifies that header `Redirect-To` exists with value `green` and routes the request to the `green` Azure function

#### "Default" traffic flow

Here is how this setup will work when I call the "default" url:

```bash
curl --get https://api.foo-bar.org/foo
```

![flow](/images/2020-10-29-active-flow.png)

* Azure Front Door receives traffic from the `api.foo-bar.org` frontend
* AFD uses `api` Routing rule
* There is no Rules engine associated with `api` Routing rule
* Request is sent to the APIM backend
* APIM policy identifies that there is no header `Redirect-To` at the request and routes the request to the active Azure function (`blue`)

Note that this solution will only work if your APIM is not deployed into private VNet or if using [External](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?WT.mc_id=AZ-MVP-5003837) deployment model.
If your APIM deployed into private VNEt with [Internal](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?WT.mc_id=AZ-MVP-5003837) mode, this solution will not work, because Azure Front Door requires endpoints to be publicly available.

I will cover how to configure the same setup with combination of API Management and Azure Application Gateway in the next post. Stay tuned!

## Useful links

* [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/?WT.mc_id=AZ-MVP-5003837#overview)
* [APIM control flow policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies?WT.mc_id=AZ-MVP-5003837#choose)
* [APIM set backend service policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies?WT.mc_id=AZ-MVP-5003837#SetBackendService)
* [Azure Front Door Rules Engine Actions](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837)
* [What is Rules Engine for Azure Front Door?](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine?WT.mc_id=AZ-MVP-5003837)
* [Modify request header](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837#modify-request-header)
* [How to use Azure API Management with virtual networks](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?WT.mc_id=AZ-MVP-5003837)

If you have any issues/comments/suggestions related to the labs, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading :)
