---
layout: post
title: "Azure DevOps Service Connection usage"
date: 2020-04-28
categories: [Azure DevOps]
---

I work a lot with [Azure DevOps](https://azure.microsoft.com/nb-no/services/devops/) and quite often I need to find what pipelines use given [Azure DevOps Service Connection](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml). From the `Azure DevOps` portal, for each service connection you can find the `Usage history`.
![Usage History](/images/2020-04-28-usage-history.png)
  That is - the list of [Releases](https://docs.microsoft.com/en-us/rest/api/azure/devops/release/releases?view=azure-devops-rest-5.1) where this service connection is used, but what I need is the list of [Release definitions](https://docs.microsoft.com/en-us/rest/api/azure/devops/release/definitions/get?view=azure-devops-rest-5.1).

Luckily, `Azure DevOps` has a variety of tools that help you fetch data that you need. First, it has a [Rest API](https://docs.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-5.1). I wrote a lot of automation scripts (both bash and python) for `Azure DevOps` using Rest API and it works really good, but scripts become a bit "noisy".

Recently, I came across [`az cli` extensions](https://docs.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview?view=azure-cli-latest) called `azure-devops`. I really like `az cli` lightweight syntax, so I decided to give it a try.

Btw, the following command gives you the list of all available extensions, in case you curious :)

```bash
az extension list-available -o table
```

Let's install `azure-devops` extensions

```bash
az extension add -n azure-devops
```

Cool, now let's see what I can do with it.

```bash
az devops --help

Group
    az devops : Manage Azure DevOps organization level operations.
        Related Groups
        az pipelines: Manage Azure Pipelines
        az boards: Manage Azure Boards
        az repos: Manage Azure Repos
        az artifacts: Manage Azure Artifacts.

Subgroups:
    admin            : Manage administration operations.
    extension        : Manage extensions.
    project          : Manage team projects.
    security         : Manage security related operations.
    service-endpoint : Manage service endpoints/service connections.
    team             : Manage teams.
    user             : Manage users.
    wiki             : Manage wikis.

Commands:
    configure        : Configure the Azure DevOps CLI or view your configuration.
    feedback         : Displays information on how to provide feedback to the Azure DevOps CLI team.
    invoke           : This command will invoke request for any DevOps area and resource. Please use
                       only json output as the response of this command is not fixed. Helpful docs -
                       https://docs.microsoft.com/en-us/rest/api/azure/devops/.
    login            : Set the credential (PAT) to use for a particular organization.
    logout           : Clear the credential for all or a particular organization.
```

quite a lot, but today I will only focus on `service-endpoint` subgroup and `az pipelines` group.

First, you need to configure your default `organization` and `project` (even though you don't need to do it, it will make your life easier later). Please find detailed [how to get started](https://docs.microsoft.com/en-us/azure/devops/cli/?view=azure-devops) documentation.

Next, you need to [sign to Azure DevOps with a Personal Access Token](https://docs.microsoft.com/en-us/azure/devops/cli/log-in-via-pat?view=azure-devops&tabs=unix).

## Get service connection id by name

When all boring but important initialization part is done, let's get some fun.
First, get the list of all service connections available.

```bash
az devops service-endpoint list -o table
```

If I want to search service connection by name, I should use `--query` parameter to [query Azure CLI command output](https://docs.microsoft.com/en-us/cli/azure/query-azure-cli?view=azure-cli-latest).

```bash
az devops service-endpoint list --query "[?name=='iac-dev-arm']"
```

where `iac-dev-arm` is the name of the service connection.

By default result will be in form of `json`, but you can use `-o` attribute to control the output format (other options are `yaml`, `table`, `jsonc` and `tsv`).

Actually, the only information I need for my task is the service connection id and I can get it with this command

```bash
az devops service-endpoint list --query "[?name=='iac-dev-arm']".id -o tsv
058c137f-37d2-4bda-9ae4-aee05568eeed
```

## Get list of all release pipeline definitions

To work with pipelines I will use `az pipelines` command group, and let's start by getting the list of all release definitions

```bash
az pipelines release definition list
```

This will return a lot of data, but I actually only need pipeline name, so I should use `--query` parameter to filter the result.

```bash
az pipelines release definition list --query [].name
```

To get release pipeline definition information, I use this command

```bash
az pipelines release definition show --name iac-dev-blue
```

where `iac-dev-blue` is the name of the pipeline.

If you analyse the release definition json or yaml data, you can see that service connection is actually stored as id

For example, here is the json fragment from one of my release definitions

```json
...
"connectedServiceNameARM": "058c137f-37d2-4bda-9ae4-aee05568eeed",
...
```

and this is the same id I got at `Get service connection id by name` section, so I am on the right track.

## The final script

Let's put it all together. We need a script that:

* gets a service endpoint name as an input parameter
* finds service connection id by the service connection name
* gets list of all release pipelines
* iterates through this list
* gets release definition for each pipeline (by name)
* search for service connection id in release definition
* if service connection is found, print pipeline name

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

Note, you need to install `jq`, the [lightweight and flexible command-line JSON processor](https://stedolan.github.io/jq/) to use final script.

and here it is in action:

```bash
$ ./get-service-connection-usage.sh iac-dev-arm
Service connection iac-dev-arm (058c137f-37d2-4bda-9ae4-aee05568eeed) is used by the following release definitions:
iac-api
```

With that - thanks for reading!
