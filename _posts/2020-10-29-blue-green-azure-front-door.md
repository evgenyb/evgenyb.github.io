---
layout: post
title: "How to do blue-green testing with Azure Front Door and API Management"
date: 2020-10-29
categories: [APIM, API Management, Canary testing, APIM policy, Azure Front Door, IaC, Infrastructure As Code]
---

![logo](/images/2020-10-29-logo.png)

If you use Azure API Management and want to adapt blue-green deployment or provisioning model, APIM has functionality that allows you implement a [canary traffic orchestration](https://borzenin.com/apim-canary-policy/) at the policy level.
Here is the typical scenario how it would work:

* you have an active version of your infrastructure provisioned and deployed to, for instance, `blue` slot
* you introduce new version into the `green` slot
* you configure APIM to send, let's say 10% of the traffic to the `green` slot
* you monitor logs and if all looks good, you increase the percentage to, let's say 50%
* eventually you sent 100% of the traffic to the `green` slot and decommission `blue`

That works fine, but sometimes you want to test your new version (in the example above that's the `green` slot) before you even open traffic for canary testing. How do you do this?

The simplest solution is to enrich requests by adding extra header with information about the `slot` we want redirect traffic to. Let's call this header `Redirect-To` with supported values `blue` or `green`. With this header at the requests, you can implement APIM policy and route the requests to the corresponding backend.

If you are lucky and have control of your APIM consumers, you can enrich requests at the client side, but often you can't do this and then the question is what options are available?

## Use Azure Front Door

The following Azure Front Door concepts will help us solve our task:

* Frontends with custom domains support
* [Rules Engine](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine?WT.mc_id=AZ-MVP-5003837)

### Additional blue|green frontends

When you create Azure Front Door, you will have at least one frontend will hostname matching the name of your Front Door instance + `.azurefd.net`. Normally, you will add a custom domain with your domain name, for instance `api.foo-bar.org` that is configured as a CNAME record, pointing to your original FD host name.  
When I want to test new inactive slot, I can add an extra endpoint with some custom domain, for example `api29cc67d2.foo-bar.org`, where `29cc67d2` is just random id, but you can use some more meaningful domain names like `api-inactive.foo-bar.org`, it doesn't really matter.

### Rules Engine

Azure Front Door has a concept of [Rules Engine](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine?WT.mc_id=AZ-MVP-5003837) that allows you to customize how HTTP requests gets handled and provides a more controlled behavior to back-ends. It supports several [actions](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837) and the one that of our interest is [Modify request header](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837#modify-request-header). This action allows you to modify headers that are present in requests sent to your origin.

Check this [tutorial](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-tutorial-rules-engine?WT.mc_id=AZ-MVP-5003837) how to configure your Rules Engine.

With rules engine I can configure the following rule:

```txt
If requestUrl contains `29cc67d2` then add new request header Redirect-To and set its value to green
```

Here is how it looks when you edit it at the Portal

![AFD-rules-engine](/images/2020-10-29-AFD-rules-engine.png)

### Routing rules

We also need to add new routing rule `api-inactive` that routes all traffic from `api29cc67d2.foo-bar.org` to `apim-backend`. This rule has to be configured to use `BlueGreenRules` engine rule.

Here is how it looks at the Portal

![routing-rule-blue](/images/2020-10-29-routing-rule-blue.png)

### APIM policies

At the API Management we need to implement a `choose` policy that checks if request contains `Redirect-To` header, and if so, extracts header value and use `set-backend-service` policy to redirect an incoming request to a corresponding beckend (either `blue` or `green`).  

### Putting it all together

With this setup in place, when I do the following request

```bash
curl --get https://api29cc67d2.foo-bar.org/foo
```

the following set actions will take place:

![flow](/images/2020-10-29-flow.png)

* Azure Front Door will receive traffic into the `api29cc67d2.foo-bar.org` frontend
* AFD will use `api-inactive` Routing rule
* AFD identifies that `api-inactive` Routing rule uses `BlueGreenRules` Rules engine
* the condition of `api-blue` engine rule are met and AFD will add `Redirect-To` header with value `green` to the request
* request is sent to the APIM backend
* APIM policy identifies that header `Redirect-To` exists with value `green` and routes the request to the `green` Azure function

## Useful links

* [Azure Front Door](https://azure.microsoft.com/en-us/services/frontdoor/?WT.mc_id=AZ-MVP-5003837#overview)
* [APIM control flow policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies?WT.mc_id=AZ-MVP-5003837#choose)
* [APIM set backend service policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies?WT.mc_id=AZ-MVP-5003837#SetBackendService)
* [Azure Front Door Rules Engine Actions](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837)
* [What is Rules Engine for Azure Front Door?](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine?WT.mc_id=AZ-MVP-5003837)
* [Modify request header](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-rules-engine-actions?WT.mc_id=AZ-MVP-5003837#modify-request-header)

If you have any issues/comments/suggestions related to the labs, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading :)
