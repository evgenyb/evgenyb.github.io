---
layout: post
title: "Declarative Azure Monitor Data Source provisioning to grafana for multi-teams setup"
date: 2020-05-18
categories: [grafana]
---

In my current project, most of the workload is running on Azure and we use Azure Monitor, Azure Log Analytics and Application Insight to collect and analyse our logs and metrics. Recently we decided that for dashboards, visualization and alerts we will use [grafana](https://grafana.com/).

Grafana supports many different [data sources](https://grafana.com/docs/grafana/latest/features/datasources/) and, by no surprise, [Azure Monitor](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/) is one of them.

The Azure Monitor data source supports the following services:

* [Azure Monitor](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#querying-the-azure-monitor-service) - the platform service that provides a single source for monitoring Azure resources.
* [Application Insights](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#querying-the-application-insights-service) - an extensible Application Performance Management (APM) service for web developers on multiple platforms.
* [Azure Log Analytics](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#querying-the-azure-log-analytics-service) gives you access to log data collected by Azure Monitor.

There are several independent autonomous teams in the project. They use different combination of Azure Monitor services to collect logs and metrics and these services are deployed to different resource groups, sometimes even in different subscriptions.

There is one grafana instance and all teams will use it. To visualize their data, teams need to configure Data Sources, corresponding to the Azure Monitor services they use. They will create a set of dashboards and alerts, based on the data from these data sources.

How should we organize grafana resources in such a multi-teams setup and how should we implement permissions model?  

## grafana terminology

Let's define some of the terms and resources used in grafana.

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

## Structuring grafana resources

Here is a list of decisions that we made in the team:

* We implemented Azure AD integration for grafana. That means that users use their Azure AD accounts to login to grafana.
* Users are assigned to one (or several) grafana Teams. We use  Azure AD team names for the grafana Teams.
* Each team has its own Dashboard folder and `folder name` = `team name`.
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

Let's check how to configure Azure Monitor data source.
Azure Monitor  data source can access metrics from different services and you can configure access to the services that you use. 

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

We want to store data source configuration as a json file stored at the repository, but we definitely don't want to store sensitive information, like Azure AD Application Client secret or Application Insight API key.
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
        "name": "iac-infra-meta-kv",
        "subscriptionId": "",
        "resourceGroupName": "iac-app-foobar-rg",
        "secretName": "appinsight-api-key"
    }
}
...
```

## Configuration file

We decided to use one file - one data source model. Here is the configuration file structure:

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

Our project uses several environments and each environment will have it's own set of Azure Monitor, Log Analytics or Application Insight services. Then team can create data source for each environment. The actual name of the data source in grafana will be: `name-environment-ds`.

### team

name of the team owning this data source

### type

we may have other type of data source. For Azure Monitor type value should be `grafana-azure-monitor-datasource`.

### monitor

If you want to add Azure Monitor service into your data source, use this section and provide the following information:

* subscriptionId - Azure Monitor subscription id
* tenantId - Azure AD tenant id
* clientId - Azure AD Application id 
* clientSecret - reference to key-vault secret containing Azure AD Application secret

### logAnalytics

If you want to add Log Analytics into your data source, use this section and provide the following information:

* subscriptionId - Azure Monitor subscription id
* tenantId - Azure AD tenant id
* workspace - Log Analytics default workspace id 
* clientId - Azure AD Application id 
* clientSecret - reference to key-vault secret containing Azure AD Application secret

### appInsights

If you want to add Application Insight into your data source, use this section and provide the following information:

* applicationId - Application Insight application id
* apiKey - reference to key-vault secret containing Application Insight API Key


## To sum-up

## Useful links

* [grafana data-source provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#datasources)
* [Data source API](https://grafana.com/docs/grafana/latest/http_api/data_source/)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
