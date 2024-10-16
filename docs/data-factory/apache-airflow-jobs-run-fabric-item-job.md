---
title: Run a Microsoft Fabric item job in Apache Airflow DAGs.
description: Learn to trigger Microsoft Fabric job item in Apache Airflow DAGs.
ms.reviewer: abnarain
ms.author: abnarain
author: abnarain
ms.topic: tutorial
ms.date: 04/15/2023
---

# Tutorial: Run a Microsoft Fabric Item Job in Apache Airflow DAGs

> [!NOTE]
> Apache Airflow job is powered by Apache Airflow. </br> [Apache Airflow](https://airflow.apache.org/) is an open-source platform used to programmatically create, schedule, and monitor complex data workflows. It allows you to define a set of tasks, called operators, that can be combined into directed acyclic graphs (DAGs) to represent data pipelines.

In this tutorial, you build a directed acyclic graph to run a Microsoft Fabric item such as Fabric Notebooks and Pipelines.

## Prerequisites

To get started, you must complete the following prerequisites:

- Enable Apache Airflow Job in your Tenant.

  > [!NOTE]
  > Since Apache Airflow job is in preview state, you need to enable it through your tenant admin. If you already see Apache Airflow Job, your tenant admin may have already enabled it.

  1. Go to Admin Portal -> Tenant Settings -> Under Microsoft Fabric -> Expand 'Users can create and use Apache Airflow Job (preview)' section.
  2. Select Apply.
     :::image type="content" source="media/apache-airflow-jobs/enable-apache-airflow-job-tenant.png" lightbox="media/apache-airflow-jobs/enable-apache-airflow-job-tenant.png" alt-text="Screenshot to enable Apache Airflow in tenant.":::

- [Create a Microsoft Entra ID app](/azure/active-directory/develop/quickstart-register-app) if you don't have one.

- Tenant level admin account must enable the setting 'Allow user consent for apps'. Refer to: [Configure user consent](/entra/identity/enterprise-apps/configure-user-consent?pivots=portal)
  :::image type="content" source="media/apache-airflow-jobs/user-consent.png" lightbox="media/apache-airflow-jobs/user-consent.png" alt-text="Screenshot to enable user consent in tenant.":::

- Add your Service principal as a "Contributor" in your Microsoft Fabric workspace.
:::image type="content" source="media/apache-airflow-jobs/manage-access.png" lightbox="media/apache-airflow-jobs/manage-access.png" alt-text="Screenshot to add service principal as a contributor.":::

- Obtain a refresh token for authentication. Follow the steps in the [Get Refresh Token](/entra/identity-platform/v2-oauth2-auth-code-flow#refresh-the-access-token) section.

- Enable the Triggerers in data workflows to allow the usage of deferrable operators.
   :::image type="content" source="media/apache-airflow-jobs/enable-triggerers.png" lightbox="media/apache-airflow-jobs/enable-triggerers.png" alt-text="Screenshot to enable triggerers.":::

## Apache Airflow plugin

To trigger an on-demand Microsoft Fabric item run, this tutorial uses the [apache-airflow-microsoft-fabric-plugin](https://pypi.org/project/apache-airflow-microsoft-fabric-plugin/) which is pre-installed in the Apache Airflow job requirements.

## Create an Airflow connection for Microsoft Fabric

1. Navigate to "View Airflow connections" to see all configured connections.
   :::image type="content" source="media/apache-airflow-jobs/view-apache-airflow-connection.png" lightbox="media/apache-airflow-jobs/view-apache-airflow-connection.png" alt-text="Screenshot to view Apache Airflow connection.":::

2. Add a new connection, using the `Generic` connection type. Configure it with:

   - <strong>Connection ID:</strong> Name of the Connection ID.
   - <strong>Connection Type:</strong> Generic
   - <strong>Login:</strong> The Client ID of your service principal.
   - <strong>Password:</strong> The refresh token fetched using Microsoft OAuth 2.0.
   - <strong>Extra:</strong> {"tenantId": The Tenant ID of your service principal.}

3. Select Save.

## Create a DAG to trigger Microsoft Fabric job items

Create a new DAG file in the 'dags' folder in Fabric managed storage with the following content. Update `workspace_id` and `item_id` with the appropriate values for your scenario:

```python
 from airflow import DAG
 from apache_airflow_microsoft_fabric_plugin.operators.fabric import FabricRunItemOperator

 with DAG(
     dag_id="Run_Fabric_Item",
     schedule_interval="@daily",
     start_date=datetime(2023, 8, 7),
     catchup=False,
 ) as dag:

     run_pipeline = FabricRunItemOperator(
         task_id="run_fabric_pipeline",
         workspace_id="<workspace_id>",
         item_id="<item_id>",
         fabric_conn_id="fabric_conn_id",
         job_type="Pipeline",
         wait_for_termination=True,
         deferrable=True,
     )

     run_pipeline
```

## Create a plugin file for the custom operator

If you want to include an external monitoring link for Microsoft Fabric item runs, create a plugin file as follows:

Create a new file in the `plugins` folder with the following content:

```python
   from airflow.plugins_manager import AirflowPlugin

   from apache_airflow_microsoft_fabric_plugin.hooks.fabric import FabricHook
   from apache_airflow_microsoft_fabric_plugin.operators.fabric import FabricRunItemLink

   class AirflowFabricPlugin(AirflowPlugin):
      """
      Microsoft Fabric plugin.
      """

      name = "fabric_plugin"
      operator_extra_links = [FabricRunItemLink()]
      hooks = [
          FabricHook,
      ]
```

## Monitor your DAG

### In Apache Airflow Job UI

1. When you open your DAG file in Fabric Managed Storage, "Results" appears at the bottom. Click on the arrow to view the results of the DAG run.
   :::image type="content" source="media/apache-airflow-jobs/monitor-in-fabric-ui.png" lightbox="media/apache-airflow-jobs/monitor-in-fabric-ui.png" alt-text="Screenshot to view Apache Airflow DAG in Apache Airflow job itself.":::

### In Apache Airflow UI

1. Go to the Airflow UI and select the DAG you created.

2. If you add the plugin, you'll see an external monitoring link. Click on it to navigate to the item run.
   :::image type="content" source="media/apache-airflow-jobs/view-apache-airflow-dags-external-link.png" lightbox="media/apache-airflow-jobs/view-apache-airflow-dags-external-link.png" alt-text="Screenshot to view Apache Airflow DAGs with external link.":::

3. Xcom Integration: Trigger the DAG to view task outputs in the Xcom tab.
   :::image type="content" source="media/apache-airflow-jobs/view-apache-airflow-dags-xcom.png" lightbox="media/apache-airflow-jobs/view-apache-airflow-dags-xcom.png" alt-text="Screenshot to view Apache Airflow DAGs with Xcom tab.":::

## Related Content

[Quickstart: Create an Apache Airflow Job](../data-factory/create-apache-airflow-jobs.md)
[Apache Airflow Job workspace settings](../data-factory/apache-airflow-jobs-workspace-settings.md)
