---
layout: post
title: "Labs from IaC Workshop: Automate Azure workload provisioning with Bicep, Powershell and Azure DevOps"
date: 2023-01-25
description: "That was an introduction level workshop that covered different aspects of workload provisioning with Bicep, PowerShell and Azure DevOps."
image: "/images/2023-01-25-logo.png"
categories: ["Bicep", "Workshops", "Infrastructure as Code", "Azure DevOps", "Azure AD", "PowerShell"]
githubissuesid: 36
---

![logo](/images/2023-01-25-logo.jpg)

Yesterday I held [Automate workload provisioning with Bicep, PowerShell and Azure DevOps](https://www.meetup.com/infrastructure-as-code-user-group-oslo/events/285739597/) workshop under my [Infrastructure as Code user group](https://www.meetup.com/Infrastructure-As-Code-User-Group-Oslo).

This time we focused on workload provisioning with `Bicep`, `PowerShell` and `Azure DevOps` and we learned:

* What is Azure Service Principal, how to create it and how to properly scope its RBAC permissions for your workload
* How to use `az devops` extension to automate Azure DevOps operations
* How to create Azure DevOps Service Connection for workload Service Principal with `az devops`
* How to automatically create IaC git Repository in Azure DevOps based on your Template repository
* How to create Azure DevOps IaC deployment pipeline using `az devops` cli

As always, labs are [available](https://github.com/evgenyb/iac-workshops/tree/main/iac-with-azure-devops) and you are welcome to work with them.

Here is IaC workshops road-map for 2023:

* Azure API Management 101
* Azure Networking 101
* AKS Workshop #7 - AKS security
* AKS Workshop #8 - Service mesh with linkerd

Don't miss any upcoming workshops and join my [Infrastructure as Code user group](https://www.meetup.com/Infrastructure-As-Code-User-Group-Oslo)!

You can also check out my previous workshops dedicated to Infrastructure as Code tools:

* [IaC workshop: Automate DNS and Certificate management on Azure with Azure DevOps](https://borzenin.com/dns-and-ssl-management-on-azure-with-ado-workshop-labs/)
* [IaC workshop: Load-Balancing Options on Azure](https://borzenin.com/azure-load-balancing-options-workshop-labs/)
* [IaC workshop: How to live in harmony with ARM templates](https://borzenin.com/iac-ws1-labs/)
* [IaC workshop: Implement immutable infrastructure on Azure with ARM templates](https://borzenin.com/iac-ws2-labs/)
* [IaC workshop: Implement immutable infrastructure with Pulumi: Part I](https://borzenin.com/iac-ws3-labs/)
* [IaC workshop: Implement immutable infrastructure with Pulumi: Part II](https://borzenin.com/iac-ws4-labs/)

and AKS

* [AKS Workshop #1 - Introduction to Azure Kubernetes Service (AKS)](https://borzenin.com/azure-kubernetes-service-aks-workshop-1-labs/)
* [AKS Workshop #2 - Advanced AKS configuration](https://borzenin.com/azure-kubernetes-service-aks-workshop-2-labs/)
* [AKS Workshop #3 - Immutable AKS infrastructure with Bicep](https://borzenin.com/azure-kubernetes-service-aks-workshop-3-labs/)
* [AKS Workshop #4 - GitOps in AKS with Flux](https://borzenin.com/azure-kubernetes-service-aks-workshop-4-labs/)
* [AKS Workshop #5 - scaling options for applications and clusters in AKS](https://borzenin.com/azure-kubernetes-service-aks-workshop-5-labs/)
* [AKS Workshop #6 - Monitoring options in AKS](https://borzenin.com/azure-aks-workshop-6-monitoring-options-aks-labs/)


With that - thanks for reading!
