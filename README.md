# Microhack 3: Advanced KQL, Policies, & Security

This Microhack is organized into the following 6 challenges:

- Challenge 7: Materialized views, Functions, External Tables
- Challenge 8: Caching and retention policies
- Challenge 9: Control commands
- Challenge 10: Cluster Auto Scaling
- Challenge 11: Security (Access control)
- Challenge 12: Security (Row level security)

Please beware that some of these challenges may not be applicable if you have deployed Azure Data Explorer cluster using the _My Free Cluster_ tier as this tier does not give you access to administer the cluster via the Azure Portal. Either upgrade the cluster to an Azure cluster (this requires an Azure Subscription), use the control commands only, or skip the challenges entirely.

---

## In order to receive the ADX microhack digital badge, you will need to complete the challenges marked with 🎓. Please submit the KQL queries/commands of these challenges in the following link: [Answer sheet - ADX Microhack 3](https://forms.office.com/r/iz4cG1ngni)

---

## Challenge 7: Materialized views, Functions, External Tables

In this challenge we will use 3 capabilities of Azure Data Explorer:

- **Materialized views** expose an aggregation query over a source table, or over another materialized view. Materialized views always return an up-to-date result of the aggregation query (always fresh). Querying a materialized view is more performant than running the aggregation directly over the source table.

- **User-defined functions** are reusable subqueries that can be defined as part of the query itself (ad-hoc functions), or persisted as part of the database metadata (stored functions). User-defined functions are invoked through a name, are provided with zero or more input arguments (which can be scalar or tabular), and produce a single value (which can be scalar or tabular) based on the function body.

- **External table** is a Kusto schema entity that references data stored outside the Azure Data Explorer database. It allows you to query data from external data stores, like Azure Blob Storage or Azure Data Lake, without ingesting it to your Azure Data Explorer cluster. The best query performance necessitates data ingestion into Azure Data Explorer. The capability to query external data without prior ingestion should only be used for historical data or data that are rarely queried.

---

### Task 1: Create a Materialized View 🎓

📆 Use table: `LogisticsTelemetry`

✍🏻 Instead of writing a query every time to retrieve the last known value for every device, create a materialized view containing the last known value for every device (the last record for each `deviceId`, based on the `enqueuedTime` column)

🕵🏻 Hint 1: Use `arg_max()`

References:

- [Materialized views](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/materialized-views/materialized-view-overview)
- [.create materialized view](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/materialized-views/materialized-view-create)
- [arg_max() aggregation function](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/arg-max-aggfunction)

---

### Task 2: Materialized views queries 🎓

✍🏻 There are two ways to query a materialized view: query the entire view or query the materialized part only. Try both of them.

References:

- [Materialized views queries](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/materialized-views/materialized-view-overview#materialized-views-queries)

---

### Task 3: User-defined Functions (Stored Functions) 🎓

✍🏻 As part of the first microhack, task 9, you wrote a query that finds out how many records start with "x", per device ID (aggregated by device ID) and rendered a pie chart. Create a stored function that will contain the code of this query. Make sure the function works.

References:

- [.create function](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/functions)

---

### Task 4: Create an External Table

✍🏻 In Microhack 1, Challenge 8, Task 3, you used the "One-click" UI (User Interface) to create a data connection to Azure Blob Storage. Use the SAS URL of `LogisticsTelemetry`, but this time you will [create an external table using the Web UI wizard](https://docs.microsoft.com/en-us/azure/data-explorer/external-table).

For more information about external tables, please refer to [External table](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/schema-entities/externaltables) and [query data in Azure Data Lake using Azure Data Explorer](https://docs.microsoft.com/en-us/azure/data-explorer/data-lake-query-data).

---

### Task 5: Querying an External Table

✍🏻 Write a query that uses the external table you created, and finds out how many device IDs start with "x".

References:

- [Querying an external table](https://docs.microsoft.com/en-us/azure/data-explorer/data-lake-query-data#querying-an-external-table)

---

---

## Challenge 8: Caching and retention policies

Among the different policies you can set to the ADX cluster, two policies are of particular importance: **retention policy** (retention period) and **cache policy** (cache period). A policy is what's used to enforce and control the properties of the cluster (or the database, or table).

The **retention** policy (grey + red box in the animation) is the time span, in days, for which it’s guaranteed that the data is kept available for querying. The time span is measured from the time that the records are ingested. When the period expires, the records will not be available for querying anymore. In other words, the retention policy defines the period in which the data available to query, measured since ingestion time. Notice that a large retention period may impact the cost.

The **cache** policy (red box in the animation), is the time span, in days, for which to keep recently ingested data (which is usually the frequently queried data) available in the hot cache rather than in long term storage (this is also known as cold cache, specifically, it is Azure Blob Storage). Data stored in the hot cache is stored in the local SSD or the RAM of the cluster's virtual machines, very close to the compute nodes. Therefore, it is more readily available for querying. The availability of data in hot cache improves query performance but can potentially increase the cluster cost (as more data is being stored, more VMs are required to store it). In other words, the caching policy defines the period in which data is kept in the hot cache before it is evicted to cold storage.

**Let's See an Animation**
This animation shows new data slotting into the database (blue bars on the right) and aged data being evited from the cache (in red) as time goes on. After data also expires from the retention policy, the data is removed from the database and thus no longer available for querying.

![Retention and Caching Policies](/assets/animations/retention-policy.gif)

All the data is always stored in the cold cache, for the duration defined in the retention policy. Any data whose age falls within the hot cache policy will also be stored in the hot cache. If you query data from cold cache, it’s recommended to target a small specific range in time ("point in time") for queries to be efficient.

---

### Task 1: Change the cache policy via the Data Explorer UI (Database-Level)

Go to your Azure Data Explorer cluster resource in the Azure portal. Click on the "Databases" blade

<img src="/assets/images/DatabasesBlade.png" width="300">

Click on the database name. The database page opens. Select "Edit" from the top menu. The side pane allows you to edit the retention and caching periods (policies) of the database. Change the retention to 365 days and the cache to 31 days, and save.

<img src="/assets/images/EditCacheAndRetention.png" width="400">
 
 ---
### Task 2: Change the cache policy via commands (database or table-level) 🎓

Database policies can be overridden per table using a KQL control command.
ADX cluster and database are Azure resources. A database is a sub-resource of the cluster, so it can be edited from the portal. Tables are not considered an Azure resource, so they cannot be managed in the portal but via a KQL command.  
You can always use KQL commands to alter the policies of the entire Cluster/Database/tables. Table level cache policy takes precedence over database level which takes precedence over cluster level.

✍🏻 Alter the cache policy of the table `LogisticsTelemetryManipulated` to 60 days.

References:

- [.alter table cache policy command](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/alter-table-cache-policy-command)

---

### Task 3: Query cold data with hot windows

Although querying cold data is possible, the data is queried faster when it's in local SSD (the hot cache), particularly for range queries that scan large amounts of data.

To query cold data, ADX processes a loading step that requires accessing a storage tier with much higher latency than the local disk. When the query is limited to a small time window, often called "point-in-time" queries, the amount of data to be retrieved will usually be small, and the query will complete quickly. For example, forensic analysts querying telemetry on a given day in the past fall under this category. The impact on the query duration depends on the size of data that is pulled from storage, and can be significant.

But, if you're scanning a large amount of cold data, query performance could benefit from using the 'hot windows' feature, which lets you efficiently query cold data.

Hot windows are part of the cache policy commands syntax and are set with the `.alter policy caching` command.

✍🏻 To try out this feature, set a hot_window between datetime(2021-01-01) .. datetime(2021-02-01)

References:

- [Use hot windows for infrequent queries over cold data in Azure Data Explorer](https://docs.microsoft.com/en-us/azure/data-explorer/hot-windows)

---

---

## Challenge 9: Control commands

### Task 1: .show/diagnostic logs/Insights

Control commands are requests to the service to retrieve information that is not necessarily data in the database tables, or to modify the service state, etc. In addition, they can be used to manage Azure Data Explorer.

The first character of the text of a request determines if the request is a control command or a query. Control commands must start with the dot (`.`) character, and no query may start by that character.

- The `.show queries` command returns a list of queries that have reached a final state, and that the user invoking the command has access to see.
- The `.show commands` command returns a table of the admin commands that have reached a final state. The TotalCpu columns is the value of the total CPU clock time (User mode + Kernel mode) consumed by this command.
- The `.show journal` command returns a table that contains information about metadata operations that are done on the Azure Data Explorer database. The metadata operations can result from a control command that a user executed, or internal control commands that the system executed, such as drop extents by retention
- The `.show tables details` command returns a set that contains the specified table or all tables in the database with a detailed summary of each table's properties.

✍🏻 Give these queries a go and learn what insights you can retrieve.

References:

- [Management (control commands) overview](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/)

---

### Task 2: Use .show queries 🎓

✍🏻 Write a command to count the number queries that you run (use the `User` column), in the past 7 day.

References:

- [.show queries](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/queries)

---

### Task 3: Use .journal commands 🎓

✍🏻 Write a command to show the details on the materialized view that you created earlier. When did you create the materialized view?

🕵🏻 Hint: use the `Event` and the `EventTimestamp` columns.

References:

- [.show journal](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/journal)

---

### Task 4: Use .show commands 🎓

✍🏻 Write a command to count the number commands that you run (use the `User` column), in the past 7 day.

References:

- [.show commands](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/commands)

---

### Task 5: Table details and size 🎓

✍🏻 Write a control command to show details on all tables in the database. How many tables are in your cluster?

🤔 What is the original size of the data, per table? What is the [extent](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/extents-overview) size of the data, per table

🕵🏻 Hint: `| extend TotalExtentSizeInMB = format_bytes(TotalExtentSize, 0, "MB")`

References:

- [.show table details](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/show-table-details-command)
- [format_bytes()](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/format-bytesfunction)

---

---

## Challenge 10: Cluster Autoscaling

Sizing a cluster appropriately is critical to the performance of Azure Data Explorer. A static cluster size can lead to under-utilization or over-utilization, neither of which is ideal. Because demand on a cluster can’t be predicted with absolute accuracy, it's better to scale a cluster, adding and removing capacity and CPU resources with changing demand.

There are two workflows for scaling an Azure Data Explorer cluster:

1. Horizontal scaling, also called "scaling in and out".
2. Vertical scaling, also called "scaling up and down".

---

### Task 1: Manage Cluster Horizontal Scaling (Scale In/Out)

By using horizontal scaling, you can scale the instance count automatically, based on predefined rules and schedules. To specify the autoscale settings for your cluster:

✍🏻 In the Azure portal, go to your Azure Data Explorer cluster resource. Under Settings, select Scale out.

✍🏻 In the Scale out window, you can select the autoscale method that you want: Manual scale, Optimized autoscale, or Custom autoscale.

Optimized autoscale is the recommended autoscale method. This method optimizes cluster performance and costs. If the cluster approaches a state of under-utilization, it will be scaled in. This action lowers costs but keeps performance level. If the cluster approaches a state of over-utilization, it will be scaled out to maintain optimal performance.

To configure Optimized autoscale: select the Optimized autoscale option. Then, select a minimum instance count and a maximum instance count. The cluster auto-scaling ranges between those two numbers, based on load.

![Scale Out](/assets/images/ScaleOut.png)

References:

- [Optimized Autoscale](https://docs.microsoft.com/en-us/azure/data-explorer/manage-cluster-horizontal-scaling#optimized-autoscale)

---

### Task 2: Manage cluster vertical scaling (scale up/down)

To Configure vertical scaling, in the Azure portal, go to your Azure Data Explorer cluster resource. Under Settings, select Scale up.

In the Scale up window, you will see a list of available SKUs for your cluster.
Please note that the vertical scaling process can take up to 30 minutes, and during that time your cluster will be suspended.

Scaling down can harm your cluster performance. Each SKU offers a distinct SSD and CPU ratio to help you correctly size their deployment and build cost optimal solutions for their enterprise analytical workload. There are four types of SKUs: types of storage optimized, compute optimized, heavy compute and isolated compute.

![Scale Up](/assets/images/ScaleUp.png)

References:

- [Choosing Cluster SKU](https://docs.microsoft.com/en-us/azure/data-explorer/manage-cluster-choose-sku)

---

---

## Challenge 11: Security (Access control)

Authorization (Cluster, Table level permissions)

Security roles define which security principals (users and applications) have permissions to operate on a secured resource such as a database or a table, and what operations are permitted. For example, principals that have the database viewer security role for a specific database can query and view all entities of that database. Managing the permissions of database table is part of the data plane management.

---

### Task 1: Security roles management 🎓

✍🏻 Run a command to list the principals that lists all security principals which have some access to the table `LogisticsTelemetryManipulated`.

References:

- [Show security roles management](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/security-roles#show-command)

---

### Task 2: Assigning roles using KQL 🎓

✍🏻 Run a command to assign a database "view" role to one of your colleagues participating in the microhack. Once the permission has been granted, ensure that the colleague has access to the table. In the following sections, we will see how you can restrict the access based on row level security.

References:

- [Managing database security roles](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/security-roles#managing-database-security-roles)
- [Principals and Identity Providers](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/access-control/principals-and-identity-providers)

---

### Task 3: Assigning roles using the Azure Portal

In addition to the control commands, the database permissions can also be set in the Azure portal.

From the cluster page, click the "Databases" blade, select the database, and click on the "Permissions" blade.

Cluster level permissions (for example, set admin permissions for _all_ the databases of the cluster) can be modified via the Azure portal (or via Azure CLI/ARM REST API). You cannot use control commands (KQL) to set cluster-level permissions.

✍🏻 To set these permissions, go to your cluster page in the Azure portal, and click on the "Permissions" blade (under 'Security + networking')

References:

- [Role based Authorization](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/access-control/role-based-authorization)

---

---

## Challenge 12: Row Level Security (RLS)

You can use Azure Active Directory group membership or principal details to control access to rows in a specific table.

✍🏻 To do so, you first create a function. This function will be later applied to the row level security (RLS) policy of your table. Once the RLS policy is enabled on a table, access is entirely replaced by the RLS function that's defined on the table. The access restriction applies to all users, including database admins and the RLS creator. The RLS query must explicitly include definitions for all types of users to whom you want to give access.

Example:

```Kusto
.create-or-alter function RLSForLogisticsTelemetry () {
  let IsInGroup = current_principal_is_member_of('aadgroup=group@fake_domain.com');
  LogisticsTelemetryManipulated | where (IsInGroup);
}
```

The following functions are often useful for row_level_security queries:

- [current_principal()](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/current-principalfunction?pivots=azuredataexplorer)
- [current_principal_details()](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/current-principal-detailsfunction)
- [current_principal_is_member_of()](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/current-principal-ismemberoffunction?pivots=azuredataexplorer)

✍🏻 Then, you enable the table's `row_level_security` policy, and use the function that will be run as part of the policy.

For example:

```Kusto
.alter table LogisticsTelemetryManipulated policy row_level_security enable "RLSForLogisticsTelemetry"
```

✍🏻 Now, try querying the table, and see whether the policy filtered the results.

```Kusto
LogisticsTelemetryManipulated
| count
```

---

### Task 1

✍🏻 Your colleague who was given access to the database in the previous challenge should no longer have access to it. Change the RLS function accordingly.

Be sure they receive no results when they query the table.

References:

- [RLS Policy](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/rowlevelsecuritypolicy)

---

🎉 Congrats! You've completed the ADX Microhack! To earn the digital badge, submit the KQL queries/commands of the challenges marked with 🎓: [Answer sheet - ADX Microhack 3](https://forms.office.com/r/iz4cG1ngni)

<details>
<summary><b>Contributing</b></summary>

This project welcomes contributions and suggestions. Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).

For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

</details>
  
<details>
<summary><b>Trademarks</b></summary>

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft trademarks or logos is subject to and must follow [Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.

</details>
