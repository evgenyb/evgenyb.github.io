---
layout: post
title: "Azure DevOps service endpoint automation"
date: 2020-06-20
categories: [ARM templates, Azure DevOps, SPN, Automation, Rest API]
---

![logo](/images/2020-06-20-logo.png)

I love automation. It gives you control, flexibility and confidence. If you use Azure DevOps pipelines for Azure provisioning and deployment, you know that Azure DevOps uses [Azure Resource Manager service connection](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops) to connect to Azure services. Azure Resource Manager service connection can be created using [automated security](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops#create-an-azure-resource-manager-service-connection-using-automated-security) or [with an existing service principal](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops#create-an-azure-resource-manager-service-connection-with-an-existing-service-principal) and both works fine if you do it from the portal. But what if you need to automate this process? What if you want to create service endpoint during provisioning or post-provisioning configuration phases?

There are 2 automation options available:

* Azure DevOps rest API
* az cli `azure-devops` extensions

Let's look at both options in details.

## Azure DevOps Rest API

You can read how to get started with Azure DevOps Rest API [here](https://docs.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-5.1) and in our case we will use [Endpoints](https://docs.microsoft.com/en-us/rest/api/azure/devops/serviceendpoint/endpoints?view=azure-devops-rest-5.1) operations.

To create new service endpoint, we need to compose a payload json describing service endpoint configuration and `POST` it to `https://dev.azure.com/{organization}/{project}/_apis/serviceendpoint/endpoints?api-version=5.1-preview.2` endpoint.

There are a lot of different service endpoint types supported by Azure Devops. The simplest way to learn the service endpoint json format, is to use [postman](https://www.postman.com/), [curl](https://linux.die.net/man/1/curl) or similar tools and get details about existing service endpoints. Alternative solution is to use Chrome Dev Tools (F12), [fiddler](https://www.telerik.com/fiddler) or similar. Since Azure DevOps portal app uses Azure DevOps rest API behind the scene, you can manually create service endpoint, analyse request / response and catch the json payload.

To communicate with Azure DevOps rest API, you need your [personal access token](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page).

My Azure DevOps organization called `evgenyborzenin`, project name is `iac-ws-2` and service endpoint id is `9880b02b-c68b-4f68-8876-d0394c44a8c1`. To get service connection details, I use the following `curl` command:

```bash
curl -u my-email:my-azure-devops-personal-access-token https://dev.azure.com/evgenyborzenin/iac-ws-2/_apis/serviceendpoint/endpoints/9880b02b-c68b-4f68-8876-d0394c44a8c1
```

Once we identified the ARM service endpoint json format, we can extract it into json template.

```json
{
  "id": "{id}",
  "name": "{service-connection-name}",
  "type": "azurerm",
  "description": "",
  "authorization": {
    "parameters": {
      "tenantid": "{tenant-id}",
      "serviceprincipalid": "{spn-id}",
      "authenticationType": "spnKey",
      "serviceprincipalkey": "{spn-secret}"
    },
    "scheme": "ServicePrincipal"
  },
  "data": {
    "subscriptionId": "{subscription-id}",
    "subscriptionName": "{subscription-name}",
    "environment": "AzureCloud",
    "scopeLevel": "Subscription",
    "creationMode": "Manual",
    "azureSpnRoleAssignmentId": "",
    "azureSpnPermissions": "",
    "spnObjectId": "",
    "appObjectId": ""
  },
  "url": "https://management.azure.com/",
  "isReady": true
}
```

For the above template, the following information needs to be provided:

| Placeholder  | description |
|---|---|
| id | Service endpoint id, -1 for new  |
| service-connection-name | Service endpoint name |
| tenant-id | Azure AD tenant id |
| spn-id | Azure AD service principal id (aka application id) |
| spn-secret | Azure AD service principal secret |
| subscription-id | Azure subscription id |
| subscription-name | Azure subscription name |

As you can see, the `authenticationType` is set to `spnKey` and that means that you need to provide spn's secret. Service principal secret is only accessible right after spn is created (you can re-create a new secret later), but you can't retrieve existing secret and that totally make sense from security stand point. Therefore, if we want to automate service endpoint creation, we either should create spn and service endpoint at the same script (so secret is assessable), or you should store spn secret to key-vault as part of the spn creation process and then get secret from the key-vault during service endpoint creation process.

With template in place, what we need to do is to replace placeholders with actual values and post json to endpoint.

Assuming that template file called `arm-service-connection-template.json` and located at the `templates` folder, here an example of how you can do replacement of the placeholders with actual values from your bash script

```bash
echo -e "Transforming template templates/arm-service-connection-template.json -> arm-service-connection.json"
cp ./templates/arm-service-connection-template.json arm-service-connection.json
sed -i -e 's/{service-connection-name}/'${serviceConnectionName}'/g' arm-service-connection.json
sed -i -e 's/{subscription-id}/'${spn_subscriptionid}'/g' arm-service-connection.json
sed -i -e 's/{subscription-name}/'"${spn_subscription_name}"'/g' arm-service-connection.json
sed -i -e 's/{spn-id}/'${spn_appid}'/g' arm-service-connection.json
sed -i -e 's/{spn-secret}/'${spn_secret}'/g' arm-service-connection.json
sed -i -e 's/{tenant-id}/'${spn_tenantid}'/g' arm-service-connection.json
```

The result of this code is `arm-service-connection.json` file with all values set.

Next, we need to identify if service endpoint already exists. The following script checks if service connection with `$serviceConnectionName` name exists and if so, gets service endpoint id.

```bash
echo -e "Check if Service Connection $serviceConnectionName already exists..."
serviceConnectionId=$(curl -s -u ${username}:${personalAccessToken} \
    -X GET "$apiUrl/_apis/serviceendpoint/endpoints?endpointNames=$serviceConnectionName&type=azurerm" \
    -H 'content-type: application/json' \
    -H 'accept: application/json;api-version=5.0-preview.2;excludeUrls=true' | jq .value[0].id -r)
```

Note, the `apiUrl` is composed as `https://dev.azure.com/{organization}/{project}`, where you need to use your own organization name and project name. In my case, I used `https://dev.azure.com/evgenyborzenin/iac-ws-2`.

If service connection exists, we want to [update existing endpoint](https://docs.microsoft.com/en-us/rest/api/azure/devops/serviceendpoint/endpoints/update%20service%20endpoint?view=azure-devops-rest-5.1), otherwise, we [create a new one](https://docs.microsoft.com/en-us/rest/api/azure/devops/serviceendpoint/endpoints/create?view=azure-devops-rest-5.1).

```bash
if [[ "$serviceConnectionId" != "null" ]]; then
    sed -i -e 's/{id}/'${serviceConnectionId}'/g' arm-service-connection.json
    echo -e "Updating Azure DevOps service connection $serviceConnectionName ($serviceConnectionId)."
    curl -s -u ${username}:${personalAccessToken}  \
        -X PUT  "$apiUrl/_apis/serviceendpoint/endpoints/$serviceConnectionId" \
        -H 'content-type: application/json' \
        -H 'accept: application/json;api-version=5.0-preview.2;excludeUrls=true' \
        --data-binary "@arm-service-connection.json" 1> /dev/null
else
    sed -i -e 's/{id}/'-1'/g' arm-service-connection.json
    resultJson=$(curl -s -u ${username}:${personalAccessToken} \
        -X POST  "$apiUrl/_apis/serviceendpoint/endpoints" \
        -H 'content-type: application/json' \
        -H 'accept: application/json;api-version=5.0-preview.2;excludeUrls=true' \
        --data-binary "@arm-service-connection.json")
    serviceConnectionId=$(echo ${resultJson} | jq .id -r)
    echo -e  "New arm Service Connection $serviceConnectionName was successfully created with id# $serviceConnectionId"
fi
```

For both create and update we use `arm-service-connection.json` file as value for curl  `--data-binary` parameter.

Last thing we need to do, is to remove `arm-service-connection.json` file.

## az cli `azure-devops`

The rest API approach worked fine for us and we used it in our project for more than a year, but recently I came across az cli [azure-devops extension](https://github.com/Azure/azure-devops-cli-extension).

It has a series of `az devops service-endpoint` related commands to get, create, update and delete service endpoints.

For example, to get service endpoint information, you can use `az devops service-endpoint show` or `az devops service-endpoint list` commands.

It also has `az devops service-endpoint azurerm create` command that creates an Azure RM type service endpoint. Let's check what parameters it requires:

```bash
az devops service-endpoint azurerm -h
Arguments
    --azure-rm-service-principal-id    [Required] : Service principal id for creating azure rm
                                                    service endpoint.
    --azure-rm-subscription-id         [Required] : Subscription id for azure rm service endpoint.
    --azure-rm-subscription-name       [Required] : Name of azure subscription for azure rm service
                                                    endpoint.
    --azure-rm-tenant-id               [Required] : Tenant id for creating azure rm service
                                                    endpoint.
    --name                             [Required] : Name of service endpoint to create.
```

Basically, you just send `--azure-rm-service-principal-id`, `--azure-rm-subscription-id`, `--azure-rm-subscription-name`, `--azure-rm-tenant-id`, `--name` parameters to the command, then you will be asked you to provide spn secret and that's it. The new ARM service endpoint is created.

We don't want to type secret manually, therefore for automation, is allows you to set service principal password/secret in `AZURE_DEVOPS_EXT_AZURE_RM_SERVICE_PRINCIPAL_KEY` environment variable. Learn more about this [here](https://aka.ms/azure-devops-cli-azurerm-service-endpoint).

## Wrapping it up

`azure-devops` extensions service endpoints functionality is still in preview, but when it will be in GA, we will definitely switch to it. With this extension, the script will be less "noisy" and much simpler because you don't need to maintain json template and don't need to do json transformation.

## Useful links

* [Service Endpoints in Azure DevOps Services](https://docs.microsoft.com/en-us/azure/devops/extend/develop/service-endpoints?view=azure-devops)
* [Manage service connections](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml)
* [Connect to Microsoft Azure](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops)
* [Azure DevOps Rest API: Endpoints - Create](https://docs.microsoft.com/en-us/rest/api/azure/devops/serviceendpoint/endpoints/create?view=azure-devops-rest-5.1)
* [Use personal access tokens](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page)
* [Creating Azure RM Service Endpoint](https://aka.ms/azure-devops-cli-azurerm-service-endpoint)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!