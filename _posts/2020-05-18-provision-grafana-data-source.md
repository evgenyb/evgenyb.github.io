---
layout: post
title: "Declarative Azure Monitor Data Source provisioning to grafana for multi-teams setup"
date: 2020-05-18
categories: [grafana]
---

In my current project, most of the workload is running on Azure and we use Azure Monitor, Azure Log Analytics and Application Insight to collect and analyse our logs and metrics. Recently we decided that for dashboards, visualization and alerts we will use [grafana](https://grafana.com/).

## grafana terminology

Let's start by introducing some of the terms and terminology used in grafana.

### Data source

Grafana supports many different storage back-ends for your time series data ([data source](https://grafana.com/docs/grafana/latest/features/datasources/#data-source-overview)). Each data source has a specific Query Editor that is customized for the features and capabilities that the particular data source exposes.

### Dashboard

A [dashboard](https://grafana.com/docs/grafana/latest/features/dashboard/dashboards/) is a set of one or more panels organized and arranged into one or more rows. Grafana ships with a variety of Panels. Each panel can interact with data from any configured Grafana [Data Source](https://grafana.com/docs/grafana/latest/features/datasources/#data-source-overview).
Dashboards can be tagged.

### Alert

[Alerts](https://grafana.com/docs/grafana/latest/alerting/alerts-overview/) allow you to identify problems in your system moments after they occur. By quickly identifying unintended changes in your system, you can minimize disruptions to your services.

### Users and Teams

[Users](https://grafana.com/docs/grafana/latest/manage-users/#users) are named accounts in Grafana that can be granted permissions to access resources throughout Grafana. 

[Teams](https://grafana.com/docs/grafana/latest/manage-users/#teams) allow you to grant permissions for a group of users.

Grafana supports many different [data sources](https://grafana.com/docs/grafana/latest/features/datasources/) and, by no surprise, [Azure Monitor](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/) is one of them.

The grafana's Azure Monitor data source supports the following services:

* [Azure Monitor](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#querying-the-azure-monitor-service) - the platform service that provides a single source for monitoring Azure resources.
* [Application Insights](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#querying-the-application-insights-service) - an extensible Application Performance Management (APM) service for web developers on multiple platforms.
* [Azure Log Analytics](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#querying-the-azure-log-analytics-service) gives you access to log data collected by Azure Monitor.

There are several independent autonomous teams in the project. They use different combination of Azure Monitor services to collect logs and metrics and these services are deployed to different resource groups, sometimes even in different subscriptions.

There is one grafana instance and all teams will use it. To visualize their data, teams need to configure Data Sources, corresponding to the Azure Monitor services they use. When data sources are in place, teams will create a set of dashboards and alerts, using data sources.

How should we organize grafana resources in such a multi-teams setup and how should we implement permissions model?  

## Structuring grafana resources

Here is a list of conventions we came up with:

* We implemented Azure AD integration for grafana. That means that users use their Azure AD accounts to login to grafana. The infrastructure architecture I will cover in one of the later posts.
* Users are assigned to one (or several) grafana Teams. We use  the same name for grafana Team as in Azure AD.
* Each team has its own Dashboard folder and with the same name as a Team name.
* We use Access Control List (ACL) model to [limit access](https://grafana.com/docs/grafana/latest/permissions/dashboard_folder_permissions/) to dashboard folder. We assign Team to the dashboard folder and team has full access (Admin) within this folder.
* We use dashboard `tags` to link dashboards to the teams.

## Team's resource provisioning

For dashboards, we are currently evaluating [grafonnet](https://grafana.github.io/grafonnet-lib/). If that works, teams will implement dashboards and deploy them to grafana with CI/CD pipelines.

When it comes to data source provisioning, There are 3 ways you can deploy data source in grafana:

1. Click-ops via grafana UI
2. Use [Data source API](https://grafana.com/docs/grafana/latest/http_api/data_source/)
3. Configure the data source with [provisioning](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#configure-the-data-source-with-provisioning)

Options 1 is not an option at all, it's a good way to "play around". Option 3 allows you to provision dashboards during grafana deployment and this is good option for post provisioning configuration. But in our case, teams should be able to configure and deploy their data sources themselves. Therefore we decided to go with declarative approach. That is - teams will configure their data sources as `json` files, create a PR to the grafana resource repository, and when PR is approved and merged, data sources will be deployed to grafana instance with CI/CD pipelines.

## Azure monitor data source configuration

Let's check what it takes to configure Azure Monitor data source.
Azure Monitor data source can access metrics from different services and each services can be configured.

### Azure Monitor service

For Azure Monitor service, you need 4 pieces of information from the Azure:

* Azure AD `Tenant id`
* Azure AD application `Client Id`
* Azure AD application `Client Secret`
* Default Subscription Id

### Azure Log Analytics service

For  Azure Log Analytics service, you need to provide the following config values:

* Azure AD `Tenant id`
* Azure AD application `Client Id`
* Azure AD application `Client Secret`
* Default Subscription Id
* Default workspace

If you use both Azure Monitor Service and Log analytics Service at the same data source and if Azure Monitor and Log Analytics are at the same subscription, , you can reuse the same configuration for Azure Log Analytics.

### Application Insights

For Application Insights you need two pieces of information:

* Application Insight Application ID
* Application Insight API Key

You can configure different combination of services at the same data source, so it can include all three, only two or one service.

## Sensitive information

We want to store data source configuration as a json file stored at git repository, but we definitely don't want to store sensitive information, like Azure AD Application Client secret or Application Insight API key.
We decided to use Azure keyvault to store sensitive information.
Since our teams use different Azure subscriptions for their resources, sensitive part of data source configuration will be distributed within key-vaults owned by the teams.

For instance, if team wants to add data source for Application Insight service, they need to specify AppInsight application ID and API Key. Since API Key is sensitive information, it will be stored to one of the team's key-vault, and then team needs to specify the following key-vault information in data source configuration:

* key-vault name
* key-vault resource group name
* key-vault subscription id
* key-vault secret name with sensitive config value

If we format this information as a json it will look like this:

```json
...
"clientSecret": {
    "keyvault": {
        "name": "iac-foobar-infra-kv",
        "subscriptionId": "00000000-0000-0000-0000-000000000000",
        "resourceGroupName": "iac-app-foobar-rg",
        "secretName": "appinsight-api-key"
    }
}
...
```

## Configuration file

We decided to use `one file - one data source` model and here is the complete configuration file structure:

```json
{
    "name": "",
    "environment": "",
    "team": "",
    "type": "grafana-azure-monitor-datasource",
    "monitor": {
        "subscriptionId": "",
        "tenantId": "",
        "clientId": "",
        "clientSecret": {
            "keyvault": {
                "name": "",
                "subscriptionId": "",
                "resourceGroupName": "",
                "secretName": ""
            }
        }
    },
    "logAnalytics": {
        "subscriptionId": "",
        "tenantId": "",
        "workspace": "",
        "clientId": "",
        "clientSecret": {
            "keyvault": {
                "name": "",
                "subscriptionId": "",
                "resourceGroupName": "",
                "secretName": ""
            }
        }
    },
    "appInsights": {
        "applicationId": "",
        "apiKey": {
            "keyvault": {
                "name": "",
                "subscriptionId": "",
                "resourceGroupName": "",
                "secretName": ""
            }
        }
    }
}
```

### name

the "logical" name of the data source

### environment

Our project uses several environments and each environment may have it's own set of Azure Monitor, Log Analytics or Application Insight services. In this case, team will need create data source for each environment. The actual name of the data source in grafana will be: `data-source-name-environment-ds`.

### team

name of the team owning data source

### type

we may have other type of data source (for example, prometheus). For Azure Monitor type, the value should be `grafana-azure-monitor-datasource`.

### monitor

If team wants to add Azure Monitor service into your data source, they need to add `monitor` section and provide the following information:

* subscriptionId - Azure Monitor subscription id
* tenantId - Azure AD tenant id
* clientId - Azure AD Application id 
* clientSecret - reference to key-vault secret containing Azure AD Application secret

### logAnalytics

If team wants to add Log Analytics into your data source, they need to add `logAnalytics` section and provide the following information:

* subscriptionId - Azure Monitor subscription id
* tenantId - Azure AD tenant id
* workspace - Log Analytics default workspace id
* clientId - Azure AD Application id 
* clientSecret - reference to key-vault secret containing Azure AD Application secret

### appInsights

If team wants to add Application Insight into your data source, they need to add `appInsights` section and provide the following configuration items:

* applicationId - Application Insight application id
* apiKey - reference to key-vault secret containing Application Insight API Key

## Provisioning script

We use bash with `jq` to work with `json`, `az cli` to work with Azure key-vault and `curl` to call grafana rest API, and here is what provisioning script will do:

* Read input parameters
* Read grafana admin user password from the key-vault
* Identify user or spn Azure AD object id
* Read json configuration from the data source file
* Parse `name`, `type` and `team` information
* If data source contains `monitor` section, parse it and extract all configuration values. Read `clientSecret` property value from the key-vault.
* If data source contains `logAnalytics` section, parse it and extract all configuration values. Read `clientSecret` property value from the key-vault.
* If data source contains `appInsights` section, parse it and extract all configuration values. Read `apiKey` property value from the key-vault.
* replace all placeholders with actual values in datasource template file 
* Check if data source already exists
* If it does, update data source
* If it doesn't, create new data source

Here is the final version of the script:

```bash
#!/usr/bin/env bash
#
# Usage ./deploy-azure-monitor-datasource.sh dev foobar.json
#
# Parameters:
#   1 - grafana instance environment 
#   2 - data source configuration file
set -euo pipefail
grafanaEnvironment=$1
datasourceFile=$2
resourceGroupName="iac-monitoring-${grafanaEnvironment}-rg"
metadataKeyvault="iac-monitoring-${grafanaEnvironment}-infra-kv"
grafanaRestAPIUrl="https://iac-monitoring-${environment}.ifoobar.no/grafana"

echo -e "Getting metadata from ${metadataKeyvault} and from Azure resources..."
adminPassword=$(az keyvault secret show --vault-name ${metadataKeyvault} -n grafana-admin-password --query value -o tsv)
usertype=$(az account show --query user.type --out tsv)

if [[ ${usertype} == "user" ]]; then
  objectId=$(az ad signed-in-user show --query "objectId" -o tsv)
else 
  objectId=$(az account show --query "id" -o tsv)
fi

echo -e "Reading data source configuration from ${datasourceFile} file"
json=$(jq -r '.' ${datasourceFile})

function parseSecretValue() {
    secretSectionJson=$1
    secretValue=""
    keyvaultName=$(echo ${secretSectionJson} | jq -r .keyvault.name)
    keyvaultSubscriptionId=$(echo ${secretSectionJson} | jq -r .keyvault.subscriptionId)
    keyvaultResourceGroupName=$(echo ${secretSectionJson} | jq -r .keyvault.resourceGroupName)
    keyvaultSecretName=$(echo ${secretSectionJson} | jq -r .keyvault.secretName)
    echo -e "Temporary add secret.get permission for ${objectId} to ${keyvaultName}(${keyvaultSubscriptionId})"
    az keyvault set-policy -n ${keyvaultName} -g ${keyvaultResourceGroupName} --object-id $objectId --secret-permissions get --subscription ${keyvaultSubscriptionId}  1> /dev/null
    echo -e "Reading ${keyvaultSecretName} secret from ${keyvaultName}(${keyvaultSubscriptionId})"
    secretValue=$(az keyvault secret show -n ${keyvaultSecretName} --vault-name ${keyvaultName} --subscription ${keyvaultSubscriptionId} --query value -o tsv)
    echo -e "Removing secret.get permission for ${objectId} from ${keyvaultName}(${keyvaultSubscriptionId})"
    az keyvault set-policy -n ${keyvaultName} -g ${keyvaultResourceGroupName} --object-id ${objectId} --secret-permissions get --subscription ${keyvaultSubscriptionId} 1> /dev/null
}

name=$(echo $json | jq -r .name)
team=$(echo $json | jq -r .team)
type=$(echo $json | jq -r .type)
environment=$(echo $json | jq -r .environment)

echo -e "Check if data source contains Monitor section"
if [[ $(echo $json | jq 'has("monitor")') == true ]]; then
    echo -e "Monitor section exists, parsing configuration"
    subscriptionId=$(echo ${json} | jq -r .monitor.subscriptionId)
    tenantId=$(echo ${json} | jq -r .monitor.tenantId)
    clientId=$(echo ${json} | jq -r .monitor.clientId)
    if [[ $(echo $json | jq .monitor | jq 'has("clientSecret")') = true ]]; then
        parseSecretValue "$(echo $json | jq .monitor.clientSecret)"
        clientSecret=${secretValue}
    else
        echo -e "No clientSecret information specified. Aborting..."
    fi
else
    echo -e "Monitor section isn't configured..."
    subscriptionId=""
    tenantId=""
    clientId=""
    clientSecret=""
fi

echo -e "Check if data source contains Log Analytics section"
if [[ $(echo $json | jq 'has("logAnalytics")') == true ]]; then
    echo -e "Log Analytics section exists, parsing configuration"
    logAnalyticsSubscriptionId=$(echo ${json} | jq -r .logAnalytics.subscriptionId)
    logAnalyticsTenantId=$(echo ${json} | jq -r .logAnalytics.tenantId)
    logAnalyticsClientId=$(echo ${json} | jq -r .logAnalytics.clientId)
    logAnalyticsDefaultWorkspace=$(echo ${json} | jq -r .logAnalytics.workspace)
    if [[ $(echo $json | jq .logAnalytics | jq 'has("clientSecret")') = true ]]; then
        parseSecretValue "$(echo $json | jq .logAnalytics.clientSecret)"
        logAnalyticsClientSecret=${secretValue}
    fi
else
    echo -e "Log Analytics section isn't configured..."
    logAnalyticsSubscriptionId=""
    logAnalyticsTenantId=""
    logAnalyticsClientId=""
    logAnalyticsClientSecret=""
    logAnalyticsDefaultWorkspace=""
fi

echo -e "Check if data source contains Application Insight section"
if [[ $(echo $json | jq 'has("appInsights")') == true ]]; then
    echo -e "Application Insight section exists, parsing configuration"
    appInsightsAppId=$(echo ${json} | jq -r .appInsights.applicationId)
    if [[ $(echo $json | jq .appInsights | jq 'has("apiKey")') = true ]]; then
        parseSecretValue "$(echo $json | jq .appInsights.apiKey)"
        appInsightsApiKey=${secretValue}
    fi
else
    echo -e "Application Insight section isn't configured..."
    appInsightsApiKey=""
    appInsightsAppId=""
fi

dataSourceName="${name}-${environment}-ds"
dataSourceId=$(curl -s -u admin:${adminPassword} \
        -X GET "$grafanaRestAPIUrl/api/datasources/name/${dataSourceName}" \
        -H 'content-type: application/json' \
        -H 'accept: application/json' | jq .id -r)
if [[ ${dataSourceId} == "null" ]]; then
    dataSourceId=0
fi

echo -e "Transforming azure-monitor-datasource-template.json to azure-monitor-datasource.json"
cat ./templates/azure-monitor-datasource-template.json | \
     sed -e 's|{dataSourceId}|'${dataSourceId}'|g' | \
     sed -e 's|{dataSourceName}|'"${dataSourceName}"'|g' | \
     sed -e 's|{clientId}|'"${clientId}"'|g' | \
     sed -e 's|{clientSecret}|'"${clientSecret}"'|g' | \
     sed -e 's|{subscriptionId}|'"${subscriptionId}"'|g' | \
     sed -e 's|{tenantId}|'"${tenantId}"'|g'  | \
     sed -e 's|{logAnalyticsDefaultWorkspace}|'"${logAnalyticsDefaultWorkspace}"'|g'  | \
     sed -e 's|{logAnalyticsClientId}|'"${logAnalyticsClientId}"'|g'  | \
     sed -e 's|{logAnalyticsClientSecret}|'"${logAnalyticsClientSecret}"'|g'  | \
     sed -e 's|{logAnalyticsSubscriptionId}|'"${logAnalyticsSubscriptionId}"'|g'  | \
     sed -e 's|{logAnalyticsTenantId}|'"${logAnalyticsTenantId}"'|g'  | \
     sed -e 's|{appInsightsAppId}|'"${appInsightsAppId}"'|g'  | \
     sed -e 's|{appInsightsApiKey}|'"${appInsightsApiKey}"'|g'  > "${dataSourceName}.json"

if [[ ${dataSourceId} == 0 ]]; then
    echo -e "Deploying new grafana Data Source ${dataSourceName} from ${dataSourceName}.json file using ${grafanaRestAPIUrl} rest API"
    dataSourceId=$(curl -s -u admin:${adminPassword} \
            -X POST "$grafanaRestAPIUrl/api/datasources" \
            -H 'content-type: application/json' \
            -H 'accept: application/json' \
            --data-binary "@${dataSourceName}.json" | jq .datasource.id -r)
else
    echo -e "Updating grafana Data Source ${dataSourceName}(${dataSourceId}) from ${dataSourceName}.json file using ${grafanaRestAPIUrl} rest API"
    dataSourceId=$(curl -s -u admin:${adminPassword} \
            -X PUT "$grafanaRestAPIUrl/api/datasources/${dataSourceId}" \
            -H 'content-type: application/json' \
            -H 'accept: application/json' \
            --data-binary "@${dataSourceName}.json" | jq .datasource.id -r)
fi

echo -e "Successfully deployed data source ${dataSourceName} - $grafanaRestAPIUrl/datasources/edit/${dataSourceId}/"

echo -e "Cleaning up..."
rm "${dataSourceName}.json"
```

and here is the data source template file

```json
{
    "id": {dataSourceId},
    "name": "{dataSourceName}",
    "type": "grafana-azure-monitor-datasource",
    "access": "proxy",
    "isDefault": false,
    "basicAuth": false,
    "jsonData": {
        "clientId": "{clientId}",
        "cloudName": "azuremonitor",
        "subscriptionId": "{subscriptionId}",
        "tenantId": "{tenantId}",
        "azureLogAnalyticsSameAs": false,
        "logAnalyticsClientId": "{logAnalyticsClientId}",
        "logAnalyticsDefaultWorkspace": "{logAnalyticsDefaultWorkspace}",
        "logAnalyticsSubscriptionId": "{logAnalyticsSubscriptionId}",
        "logAnalyticsTenantId": "{logAnalyticsTenantId}",
        "appInsightsAppId": "{appInsightsAppId}"
    },
    "secureJsonData": {
        "clientSecret": "{clientSecret}",
        "logAnalyticsClientSecret": "{logAnalyticsClientSecret}",
        "appInsightsApiKey": "{appInsightsApiKey}"
    }
}
```

## To sum-up

Theis is 

## Useful links

* [grafana data-source provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#datasources)
* [Data source API](https://grafana.com/docs/grafana/latest/http_api/data_source/)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
