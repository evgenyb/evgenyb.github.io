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

* We implemented Azure AD integration for grafana, that means that users use their Azure AD accounts to login to grafana.
* Users are assigned to one (or several) Teams. We use Azure AD team name for grafana Teams.
* Each team has its own Dashboard folder and `folder name` = `team name`.
* We use Access Control List (ACL) model to [limit access](https://grafana.com/docs/grafana/latest/permissions/dashboard_folder_permissions/) to dashboard folder. We assign Team to the dashboard folder and team has full access (Admin) within this folder.
* We use dashboard `tags` to link dashboards to the teams.

## Team's resource provisioning

For dashboards, we are currently evaluating [grafonnet](https://grafana.github.io/grafonnet-lib/). If that works, teams will implement dashboards and deploy them to grafana with CI/CD pipelines.

There are 3 ways you can deploy data source in grafana:

* Click-ops via grafana UI
* Use [Data source API](https://grafana.com/docs/grafana/latest/http_api/data_source/)
* Configure the data source with [provisioning](https://grafana.com/docs/grafana/latest/features/datasources/azuremonitor/#configure-the-data-source-with-provisioning)

For Azure monitor we decided to go with declarative approach. That is - teams will configure their data sources as `json` files, create a PR to the grafana resource repository and when approved and merged, they will be deployed to grafana with CI/CD pipelines.

## To sum-up

## Useful links

* [grafana data-source provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#datasources)
* [Data source API](https://grafana.com/docs/grafana/latest/http_api/data_source/)

If you have any issues/comments/suggestions related to this post, you can reach out to me at evgeny.borzenin@gmail.com.

With that - thanks for reading!
