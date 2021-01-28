---
layout: post
title: "How to use Kubernetes ExternalName Service type to migrate your applications to different namespaces with zero downtime."
date: 2021-01-28
description: "It's easy to reorganize your Kubernetes namespaces structure in test environments, but you need to have a solid plan how to migrate your applications between namespaces in your production cluster with zero downtime. In this blogpost I show how you can use Kubernetes ExternalName Service type to implement proxy-services to achieve zero downtime migration."
image: "/images/2021-01-27-step5.png"
categories: ["AKS", "Kubernetes"]
githubissuesid: 28
---

At my current project, we have started restructuring our cluster's namespaces model. We are moving from `one namespace` to `namespace per team/domain` model. 
One of the challenges that we immediately encountered was that when you move application to a different namespace, you need to change service URLs of all dependent applications and include namespace. Here is an abstracted example:

There are 3 applications `AppA`, `AppB` and `AppC`. All deployed with corresponding Kubernetes services called `ServiceA`, `ServiceB` and `ServiceC`.
All 3 applications are deployed to the same namespace called `foobar`. `AppA` has inbound dependency from `AppC` and outbound dependency to `AppB`. 

![foobar](/images/2021-01-27-foobar.png)

Since everything is deployed to the same namespace, we use the following URLs to call our services:

* `http://servicea/` 
* `http://serviceb/`
* `http://servicec/` 
 
Now, we want to move `AppA` to the new `foo` namespace.

![step1](/images/2021-01-27-step1.png)

Since now applications are deployed to different namespaces, we need to change and use the following URLs to call services:

* `http://servicea.foo.svc.cluster.local/` 
* `http://serviceb.foobar.svc.cluster.local/`
* `http://servicec.foobar.svc.cluster.local/` 

Not only that requires changes at all 3 applications, but also may cause a possible down time at `AppC` during the migration period. In real life, with hundreds of applications migrating to several namespaces, that can cause even longer down time and depending on the dependency graph, can require a lot of deployment orchestration during transition period. Not to mention that it might be that some of the applications can't be re-configured and re-deployed at the moment. 
All these factors forced us to think how we can do such a migration with close to zero down time. 

## Proxy with ExternalName Service type

The solution that we came up with was to introduce a proxy-service at the old namespace with the same name as service being migrated, pointing to the service in the new namespace.

![proxy](/images/2021-01-27-final.png)

The proxy-service `ServiceA` is deployed to the `foobar` namespace and routes traffic to the "real" `ServiceA` at the `foo` namespace. Because proxy-service deployed to the same namespace, `AppC` can still call it by using `http://servicea/`.

Kubernetes documentation describes services of type [ExternalName](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) as Service that maps a Service to a DNS name, not to a typical selector. 
Here is a proxy-service definition for `servicea` at `foobar` namespace:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: servicea
  namespace: foobar
spec:
  type: ExternalName
  externalName: servicea.foo.svc.cluster.local
```

## Migration plan

With proxy-service in place, here is our migration strategy:

1. Re-configure `AppA` with correct URLs for all outbound dependencies 
2. Deploy `AppA` into `foo` namespace
![step2](/images/2021-01-27-step2.png)
3. Replace `servicea` in `foobar` namespace with proxy-service
![step3](/images/2021-01-27-step3.png)
4. Delete `AppA` pods (and other related Kubernetes resources) from `foobar` namespace
![step4](/images/2021-01-27-step4.png)
5. Deploy new version of `AppC` with URL pointing to `servica` at `foo` namespace
![step5](/images/2021-01-27-step5.png)
6. When all applications calling `AppA` at `foobar` namespace are re-configured and re-deployed, delete `servicea` from `foobar` namespace
![step6](/images/2021-01-27-step1.png)


## Useful links

* [Kubernetes service](https://kubernetes.io/docs/concepts/services-networking/service/)
* [Services without selectors](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)
* [Type ExternalName](https://kubernetes.io/docs/concepts/services-networking/service/#externalname)

With that - thanks for reading :)
