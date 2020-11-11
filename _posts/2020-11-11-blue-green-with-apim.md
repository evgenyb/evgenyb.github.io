---
layout: post
title: "How to do blue-green testing with API Management"
date: 2020-11-11
description: "If you use Azure API Management and want to adapt a immutable infrastructure with blue-green provisioning model, API Management has functionality that allows you to implement a canary traffic orchestration at the policy level. The canary testing model works fine, but sometimes you need to verify a new version of your infrastructure before you open traffic even for canary testing. How do you do this? In this blogpost I show how you can orchestrate your traffic to an inactive version of infrastructure just by using API Management."
image: "/images/2020-11-11-logo.png"
categories: ["API Management", "Canary testing", "Infrastructure as Code", "Immutable Infrastructure"]
githubissuesid: 23
---

If you use Azure API Management and want to adapt blue-green provisioning model, API Management has functionality that allows you to implement a [canary traffic orchestration](https://borzenin.com/apim-canary-policy/) at the policy level. In my previous posts I covered:

* [How to do blue-green testing with Azure Front Door and API Management](https://borzenin.com/blue-green-azure-front-door/)
* [How to do blue-green testing with API Management and Azure Application Gateway](https://borzenin.com/blue-green-azure-application-gateway/)

![logo](/images/2020-11-11-logo.png)

Here is a typical scenario for how blue-green infrastructure provisioning works:

* you have an active version of your infrastructure provisioned and deployed to a slot that we will call the `blue` slot
* you introduce a new version of your infrastructure and deploy it into a new slot - the `green` slot
* you configure APIM to send some small percentage (let's say 10%) of the traffic to the `green` slot
* you monitor logs and if all looks good, you switch all 100% of the traffic to the `green` slot and decommission the `blue` one

Now, I want to test my new version (the `green` slot) before I open traffic for canary testing. How do I do that?

The two previous use-cases required additional infrastructure components, like Azure Front Door or Application Gateway, which you only have to add for higher value it provides, such as traffic acceleration (Azure Front Door) and WAF (Azure Front Door and Application Gateway). If you don't need any of this, you can still achieve the same result just by using API Management functionality and that's what we will cover in this post.

## Use APIM custom domain

When you create an Azure API Management service instance, Azure assigns a subdomain of azure-api.net to it (for example, `foobar-apim.azure-api.net`). Normally, in addition to the default domain, you can configure and expose your API Management endpoint using your custom domain name, such as `api.foo-bar.com`.

In my case, when I need to test the inactive infrastructure slot, I can configure an extra domain for APIM Gateway endpoint, for example `api29cc67d2.foo-bar.org`. With custom domain in place, I can implement [<choose...> (aka Control flow policy)](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies?WT.mc_id=AZ-MVP-5003837#choose) and route all traffic coming from `api29cc67d2.foo-bar.com` host to the inactive backend.

Here is an implementation example:

```xml
{% raw %}
<inbound>
    ...
    <choose>
        <when condition="@(context.Request.OriginalUrl.Host.Contains("api29cc67d2.foo-bar.com"))">
            <set-backend-service base-url="{{inactiveBackendUrl}}" />
        </when>
        <otherwise>
            <set-backend-service base-url="{{activeBackendUrl}}" />
        </otherwise>
    </choose>
    ...
</inbound>
{% endraw %}
```

where `inactiveBackendUrl` and `activeBackendUrl` are Named values, representing active and inactive back-end urls. As I described in [this post](https://borzenin.com/apim-canary-policy/), if you have a lot of APIs, then it might be a better solution to "compose" backend Url at the Global or Product level policy, store it into the context level variable and use it at the `<set-backend-service...>` at the API or operation level policy. This way, you orchestration logic will be stored at one place only.  

## Use Revisions

Another option is to use [revisions](https://docs.microsoft.com/en-us/azure/api-management/api-management-revisions?WT.mc_id=AZ-MVP-5003837). Revisions allow you to make changes to your APIs in a controlled and safe way. Each revision to your API can be accessed using a specially formed URL. Append `;rev={revisionNumber}` at the end of your API URL, but before the query string, to access a specific revision of that API. For example, I might use this URL to access revision 1 of my `foo` API:

```bash
https://api.foo-bar.org/foo;rev=2
```

Then I can implement [<choose...> (aka Control flow policy)](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies?WT.mc_id=AZ-MVP-5003837#choose) and route all traffic containing `rev=` as prt of the url to the inactive backend.

```xml
{% raw %}
<inbound>
    ...
    <choose>
        <when condition="@(context.Request.OriginalUrl.Url.Contains("rev="))">
            <set-backend-service base-url="{{inactiveBackendUrl}}" />
        </when>
        <otherwise>
            <set-backend-service base-url="{{activeBackendUrl}}" />
        </otherwise>
    </choose>
    ...
</inbound>
{% endraw %}
```

With this approach, when I want to test my inactive infrastructure, I need to create a new set of revisions for each API under test and use revision URLs in my tests.

## Useful links

* [About API Management](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts?WT.mc_id=AZ-MVP-5003837)
* [Azure API Management terminology](https://docs.microsoft.com/en-us/azure/api-management/api-management-terminolog?WT.mc_id=AZ-MVP-5003837)
* [API Management documentation](https://docs.microsoft.com/en-us/azure/api-management?WT.mc_id=AZ-MVP-5003837)
* [Configure a custom domain name for your Azure API Management instance](https://docs.microsoft.com/en-us/azure/api-management/configure-custom-domain?WT.mc_id=AZ-MVP-5003837)
* [Policies in Azure API Management](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-policies?WT.mc_id=AZ-MVP-5003837)
* [API Management policy expressions](https://docs.microsoft.com/en-us/azure/api-management/api-management-policy-expressions?WT.mc_id=AZ-MVP-5003837)
* [APIM policy context variable](https://docs.microsoft.com/en-us/azure/api-management/api-management-policy-expressions?WT.mc_id=AZ-MVP-5003837#ContextVariables)
* [API Management policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-policies?WT.mc_id=AZ-MVP-5003837)
* [APIM control flow policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies?WT.mc_id=AZ-MVP-5003837#choose)
* [APIM set backend service policy](https://docs.microsoft.com/en-us/azure/api-management/api-management-transformation-policies?WT.mc_id=AZ-MVP-5003837#SetBackendService)
* [How to use Azure API Management with virtual networks](https://docs.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?WT.mc_id=AZ-MVP-5003837)
* [Revisions in Azure API Management](https://docs.microsoft.com/en-us/azure/api-management/api-management-revisions?WT.mc_id=AZ-MVP-5003837)

With that - thanks for reading :)
