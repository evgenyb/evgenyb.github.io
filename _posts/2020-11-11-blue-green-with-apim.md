---
layout: post
title: "How to do blue-green testing with API Management"
date: 2020-11-06
description: "If you use Azure API Management and want to adapt a blue-green provisioning model, API Management has functionality that allows you to implement a canary traffic orchestration at the policy level. The canary testing model works fine, but sometimes you need to verify your new version of the infrastructure before you open traffic even for canary testing. How do you do this? In this blogpost I show how you can orchestrate your traffic to an inactive version of infrastructure just by using API Management."
image: "/images/2020-11-11-logo.png"
categories: ["API Management", "Canary testing", "Infrastructure as Code", "Immutable Infrastructure"]
---

If you use Azure API Management and want to adapt blue-green provisioning model, API Management has functionality that allows you to implement a [canary traffic orchestration](https://borzenin.com/apim-canary-policy/) at the policy level.

![logo](/images/2020-11-11-logo.png)

This is the third post in the serious of posts describing how to test inactive infrastructure before you open traffic into it. In my previous posts I covered:

* [How to do blue-green testing with Azure Front Door and API Management](https://borzenin.com/blue-green-azure-front-door/)
* [How to do blue-green testing with API Management and Azure Application Gateway](https://borzenin.com/blue-green-azure-application-gateway/)

Again, here is a typical scenario for how blue-green infrastructure provisioning works:

* you have an active version of your infrastructure provisioned and deployed to a slot that we will call the `blue` slot
* you introduce a new version of your infrastructure and deploy it into a new slot - the `green` slot
* you configure APIM to send some small percentage (let's say 10%) of the traffic to the `green` slot
* you monitor logs and if all looks good, you switch all 100% of the traffic to the `green` slot and decommission the `blue` one

Now, I want to test my new version (the `green` slot) before I open traffic for canary testing. How do I do that?

The two previous use-cases required additional infrastructure components, like Azure Front Door or Application Gateway, which you only have to add for higher value it provides, such as traffic acceleration (Azure Front Door) and WAF (Azure Front Door and Application Gateway). Actually, you can achieve the same result just by using API Management functionality and that's what we will cover in this post.

## Configure APIM custom domain

When you create an Azure API Management service instance, Azure assigns it a subdomain of azure-api.net (for example, `foobar-apim.azure-api.net`). Normally, in addition to the default domain, you can expose your API Management endpoints using your domain name, such as `apim.foo-bar.com`.

When I need to test the new, inactive infrastructure slot, I can add new custom domain for APIM Gateway endpoint

## Useful links

* [About API Management](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts?WT.mc_id=AZ-MVP-5003837)
* [Azure API Management terminology](https://docs.microsoft.com/en-us/azure/api-management/api-management-terminolog?WT.mc_id=AZ-MVP-5003837)
* [API Management documentation](https://docs.microsoft.com/en-us/azure/api-management?WT.mc_id=AZ-MVP-5003837)
* [Configure a custom domain name for your Azure API Management instance](https://docs.microsoft.com/en-us/azure/api-management/configure-custom-domain?WT.mc_id=AZ-MVP-5003837)
* [Policies in Azure API Management](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-policies?WT.mc_id=AZ-MVP-5003837)
* [API Management policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-policies?WT.mc_id=AZ-MVP-5003837)
* [APIM control flow policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies?WT.mc_id=AZ-MVP-5003837#choose)
* [APIM set backend service policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies?WT.mc_id=AZ-MVP-5003837#SetBackendService)
* [How to use Azure API Management with virtual networks](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?WT.mc_id=AZ-MVP-5003837)

With that - thanks for reading :)
