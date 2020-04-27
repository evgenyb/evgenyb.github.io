---
layout: post
title: "Azure DevOps Service Connection usage"
date: 2020-04-28
categories: [Azure DevOps]
---

I work a lot with [Azure DevOps](https://azure.microsoft.com/nb-no/services/devops/) and quite often I need to find what pipelines use given [Azure Service Connection](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml). From the `Azure DevOps` portal you can find the `Usage history`, that is - the list of [Releases](https://docs.microsoft.com/en-us/rest/api/azure/devops/release/releases?view=azure-devops-rest-5.1) where this service connection is used. What we need is the list of [Release definitions](https://docs.microsoft.com/en-us/rest/api/azure/devops/release/definitions/get?view=azure-devops-rest-5.1).

Luckily, `Azure DevOps` has a reach tool that helps you fetch data you need. First, it has well documented [Rest API](https://docs.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-5.1), but working with it is a little bit noisy. Recently I came across [`az cli` extensions](https://docs.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview?view=azure-cli-latest) called `azure-devops`.

Tip. The following command gives you the list of available extensions.

```bash
az extension list-available -o table
```

Let's install `azure-devops` extensions.

```bash
az extension add -n azure-devops
```

Cool, now let's see what we can do with it.

```bash
az devops --help

Group
    az devops : Manage Azure DevOps organization level operations.
        Related Groups
        az pipelines: Manage Azure Pipelines
        az boards: Manage Azure Boards
        az repos: Manage Azure Repos
        az artifacts: Manage Azure Artifacts.
```

You can read how to get started with az devops [here](https://docs.microsoft.com/en-us/azure/devops/cli/?view=azure-devops).

Next, you need to [sign to Azure DevOps with a Personal Access Token](https://docs.microsoft.com/en-us/azure/devops/cli/log-in-via-pat?view=azure-devops&tabs=unix)

## Get service connection by name

When all boring but important initialization part is done, let's get some fun.

```bash
az devops service-endpoint list --query "[?name=='iac-dev-arm']"
```

where `iac-dev-arm` is the name of the service connection.

By default result will be in form of `json`, but you can use `-o` attribute to control the output format (these are the other options available `yaml`, `table`, `jsonc` `tsv`).

Actually, the only information we need for our task is the service connection id and we can get it with this command

```bash
az devops service-endpoint list --query "[?name=='iac-dev-arm']".id | jq .[0] -r
```

## Get list of all release pipeline definitions

WThis time can should use `pipelines` command group 

```bash
az pipelines release definition list
```

This will return a lot of data, but we actually only need pipeline name, so we can use `--query` argument to construct the result.

The pipeline definition list doesn't contain information about scrive endpoint, therefore the only information we need from this list is release definition name, so, let's correct our command  

```bash
az pipelines release definition list --query [].name
```

To get release pipeline definition information we can use this command

```bash
az pipelines release definition show --name iac-dev-blue
```

where `iac-dev-blue` is the name of the pipeline.

If you analyse the release definition json or yaml data, you can see that service connection is actually stored as id

For example, here is the json fragment from my release definition json

```json
...
"connectedServiceNameARM": "e71d5aa5-0458-4a19-8f72-cd4969d69c02",
...
```

## Final script

Let's put it all together. We need a script that:

* gets a service endpoint name as an input parameter
* finds service connection id by the service connection name
* gets list of all release pipelines
* iterates through this list
* gets release definition for each pipeline by name
* search for service connection id in release definition
* if service connection is used, print pipeline name

And here is the final version:

```bash
#!/usr/bin/env bash
# Finds all release definitions that use given service connection.
#
# usage: ./get-service-connection-usage.sh iac-dev-arm
#

serviceConnectionName=$1

serviceConnectionId=$(az devops service-endpoint list --query "[?name=='$serviceConnectionName']".id | jq .[0] -r)
echo -e "Service connection $serviceConnectionName ($serviceConnectionId) is used by the following release definitions:"

az pipelines release definition list --query [].name | jq -r .[] | while read name; do
    responseJson=$(az pipelines release definition show --name $name) 
    containsServiceEndpoint=$(echo $responseJson | grep $serviceConnectionId) 
    if [[ "${containsServiceEndpoint}" != "" ]]; then
        name=$(echo ${responseJson} | jq .name -r)
        echo -e "$name"
    fi
done
```

With that - thanks for reading!
