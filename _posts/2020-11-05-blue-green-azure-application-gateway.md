---
layout: post
title: "How to do blue-green testing with API Management and Azure Application Gateway"
date: 2020-11-05
description: "If you use Azure API Management and want to adapt a blue-green provisioning model, APIM has functionality that allows you to implement a canary traffic orchestration at the policy level. The canary testing model works fine, but sometimes you need to verify your new version of the infrastructure before you open traffic even for canary testing. How do you do this? In this blogpost I show how you can orchestrate your traffic to an inactive version of infrastructure by using API Management in combination with Azure Application Gateway."
image: "/images/2020-11-05-logo.png"
categories: [APIM, API Management, Canary testing, APIM policy, Azure Application Gateway, AGW, IaC, Infrastructure As Code]
---

![logo](/images/2020-11-05-logo.png)

If you use Azure API Management and want to adapt blue-green deployment or provisioning model, APIM has functionality that allows you to implement a [canary traffic orchestration](https://borzenin.com/apim-canary-policy/) at the policy level.

In my previous post I wrote about [how to do blue-green testing with Azure Front Door and API Management](https://borzenin.com/blue-green-azure-front-door/), but I will repeat the use-case description and requirements here as well.

Here is a typical scenario for how blue-green infrastructure provisioning works:

* you have an active version of your infrastructure provisioned and deployed to a slot that we will call the `blue` slot
* you introduce a new version of your infrastructure and deploy it into a new slot - the `green` slot
* you configure APIM to send some small percentage (let's say 10%) of the traffic to the `green` slot
* you monitor logs and if all looks good, you switch all 100% of the traffic to the `green` slot and decommission the `blue` one

Now, I want to test my new version (the `green` slot) before I open traffic for canary testing. How do I do that?

In this post I will focus on the use-case when APIM is deployed to a private VNet with [Internal](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?WT.mc_id=AZ-MVP-5003837) mode and exposed publicly with Azure Application Gateway.

## Application Gateway Configuration

Here is what you typically need to configure at Application Gateway:

### Http Listeners

A [listener](https://docs.microsoft.com/en-us/azure/application-gateway/configuration-listeners?WT.mc_id=AZ-MVP-5003837) is a logical entity that checks for incoming connection requests by using the port, protocol, host, and IP address. In my example, there is one listener called `default` configured as a custom domain `api.foo-bar.org`.

### Backend pools

A [backend pool](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-components?WT.mc_id=AZ-MVP-5003837#backend-pools) routes request to backend servers, which serve the request. In my example, there is one backend called `apim` pointing to APIM instance.

### Routing rules

A [request routing rule](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-components?WT.mc_id=AZ-MVP-5003837#request-routing-rules) determines how to route traffic on the listener. The rule binds the listener and the back-end pool. In my example, there is a rule called `default` that links `default` listener with `apim` backend pool.

Here is how AGW configuration looks like from the `Monitoring/Insights` view (still in Preview)

![AGW-default](/images/2020-11-05-AGW-default.png)

## Canary configuration

AGW allows you to configure more than one listener, routing rule and backend pool, so, when I need to test the new inactive infrastructure slot, I can add an extra listener, let's call it `canary`, receiving requests from the test host `api29cc67d2.foo-bar.org`, and new routing rule that binds `canary` listener with `apim` backend pool, and let's call it `canary` as well.

With this new set of components in place, here is how AGW configuration will look like:

![AGW-default](/images/2020-11-05-AGW-canary.png)

## Rewrite HTTP headers

Application Gateway supports [rewrite HTTP headers of requests and responses](https://docs.microsoft.com/en-us/azure/application-gateway/rewrite-http-headers-url?WT.mc_id=AZ-MVP-5003837). It allows you to add conditions to ensure that specified headers are rewritten only when certain conditions are met.
Application Gateway uses [server variables](https://docs.microsoft.com/en-us/azure/application-gateway/rewrite-http-headers-url?WT.mc_id=AZ-MVP-5003837#server-variables) to store useful information about the server, the connection with the client, and the current request on the connection. You can use these variables to evaluate rewrite conditions and rewrite headers.

For my case, I want to enrich all requests coming through `canary` routing rules with extra header `Redirect-To` with value set to `green`.

This is how Rewrite set configuration looks at the Portal:

First, select what Routing rules that you want rewrite set to be associated with. In my case, I only want to enrich request headers for `canary` routing rule.

![portal-01](/images/2020-11-05-portal-1.png)

Next, configure your rewrite set. I want to enrich all requests coming through `canary` routing rules by adding new `Redirect-To` header before AGW forwards the requests to the backend, therefore there is no Conditions and only one Action called `CanaryRewrite` that adds new `Redirect-To` header with value `green`.

![portal-02](/images/2020-11-05-portal-2.png)

Here is how the AGW configuration will look like when rewrite set is assigned to the `canary` routing rule:

![AGW-canary-v1](/images/2020-11-05-AGW-canary-v1.png)

## APIM policies

At the API Management side I need to implement a `choose` policy that checks if requests contain `Redirect-To` header, and if so, use `set-backend-service` policy to redirect the incoming request to the backend hosted at the `green` slot.  

### Putting it all together

#### "Default" traffic flow

Here is how this setup will work when I call the "default" url:

```bash
curl --get https://api.foo-bar.org/foo
```

![flow](/images/2020-11-05-active-flow.png)

* Application Gateway receives traffic from the `api.foo-bar.org` listener
* AGW uses `default` routing rule
* There is no Rewrite set associated with `default` routing rule
* Request is sent to the APIM backend
* APIM policy identifies that there is no header `Redirect-To` at the request and routes the request to the App Service hosted at the active slot (`blue`)

#### Traffic to inactive slot

When I call "inactive" frontend

```bash
curl --get https://api29cc67d2.foo-bar.org/foo
```

the following set of actions will take place:

![flow](/images/2020-11-05-inactive-flow.png)

* Application Gateway receives traffic from the `api29cc67d2.foo-bar.org` listener
* AGW uses `canary` routing rule
* AGW identifies that `canary` routing rule has `CanaryRewrite` rewrite set assigned
* AGW will add `Redirect-To` header with value `green` to the request
* Request is sent to the APIM backend
* APIM policy identifies that header `Redirect-To` exists with value `green` and routes the request to the `green` App Service

Next time I will describe how to test inactive slot just by using the built-in APIM functionality. Stay tuned!

## Useful links

* [What is Azure Application Gateway?](https://docs.microsoft.com/en-us/azure/application-gateway/overview?WT.mc_id=AZ-MVP-5003837)
* [Azure Application Gateway documentation](https://docs.microsoft.com/en-us/azure/application-gateway/?WT.mc_id=AZ-MVP-5003837)
* [Application gateway components](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-components?WT.mc_id=AZ-MVP-5003837)
* [Application Gateway configuration overview](https://docs.microsoft.com/en-us/azure/application-gateway/configuration-overview?WT.mc_id=AZ-MVP-5003837)
* [AGW: listeners](https://docs.microsoft.com/en-us/azure/application-gateway/configuration-listeners?WT.mc_id=AZ-MVP-5003837)
* [AGW: backend pool](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-components?WT.mc_id=AZ-MVP-5003837#backend-pools)
* [AGW: request routing rule](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-components?WT.mc_id=AZ-MVP-5003837#request-routing-rules)
* [AGW: Rewrite HTTP headers of requests and responses](https://docs.microsoft.com/en-us/azure/application-gateway/rewrite-http-headers-url?WT.mc_id=AZ-MVP-5003837)
* [APIM control flow policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies?WT.mc_id=AZ-MVP-5003837#choose)
* [APIM set backend service policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies?WT.mc_id=AZ-MVP-5003837#SetBackendService)
* [How to use Azure API Management with virtual networks](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?WT.mc_id=AZ-MVP-5003837)

With that - thanks for reading :)

## Responses

Visit the [Github Issue](https://github.com/evgenyb/evgenyb.github.io/issues/21) to comment on this page. The comments will not be displayed directly on that page.
