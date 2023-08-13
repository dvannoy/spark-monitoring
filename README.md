# Monitoring Azure Databricks in an Azure Log Analytics Workspace

|   |  |
|-----------|:--------------------------|
| :exclamation: | As of **May 10th, 2023** This library now supports Azure Databricks Runtimes 11.0 and above (see [Supported configurations](#supported-configurations)). Please note the 11.0 release is not backwards compatible due to the different logging systems used in the Databricks Runtimes. Be sure to use the correct build for your Databricks Runtime.|
| :warning: | This library and GitHub repository are in *maintenance mode*. There are no plans for further releases, and issue support will be best-effort only. For any additional questions regarding this library or the roadmap for monitoring and logging of your Azure Databricks environments, please contact [azure-spark-monitoring-help@databricks.com](mailto:azure-spark-monitoring-help@databricks.com). |
|   |  |

This repository extends the core monitoring functionality of Azure Databricks to send streaming query event information to Azure Monitor. For more information about using this library to monitor Azure Databricks, see [Monitoring Azure Databricks](https://docs.microsoft.com/azure/architecture/databricks-monitoring)

The project has the following directory structure:

```shell
/src
/pom.xml
/sample
    /spark-sample-job
/perftools
     /spark-sample-job
```

The **src** directory contains the code for building the JAR file that are deployed to the Databricks cluster. The **src** directory includes a **scripts** directory that contains a cluster node initialization script to copy the JAR file from a staging directory in the Azure Databricks file system to execution nodes. The **pom.xml** file is the main Maven project object model build file for the entire project.

The **spark-sample-job** directory is a sample Spark application demonstrating how to implement a Spark application metric counter.

The **perftools** directory contains details on how to use Azure Monitor with Grafana to monitor Spark performance.

## Prerequisites

Before you begin, ensure you have the following prerequisites in place:

* Clone this GitHub repository and switch to this branch (`git switch l4jv2`).
* An active Azure Databricks workspace. For instructions on how to deploy an Azure Databricks workspace, see [get started with Azure Databricks.](https://learn.microsoft.com/en-us/azure/databricks/getting-started/#use-the-portal-to-create-an-azure-databricks-workspace).
* Install the [Azure Databricks CLI](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/cli/databricks-cli#install-the-cli).
  * Authentication is reqiured to use the CLI. Set up one of the unified authentication methods, such as Azure Databricks personal access token authentication. For instructions, see [Set up authentication](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/cli/databricks-cli#--set-up-authentication).
  * You can also use the Azure Databricks CLI from the Azure Cloud Shell.
* A Java IDE, with the following resources:
  * [Java Development Kit (JDK) version 1.8](https://www.oracle.com/technetwork/java/javase/downloads/index.html)
  * [Scala language SDK 2.12](https://www.scala-lang.org/download/)
  * [Apache Maven 3.6.3](https://maven.apache.org/download.html)

### Supported configurations

| Databricks Runtime(s) | Maven Profile |
| -- | -- |
| `11.3LTS` | `dbr-11.3-lts` |  
| `12.2LTS` | `dbr-12.2-lts` |  

To add a new DBR versions, add a new profile in the pom.xml file

## Logging Event Size Limit

This library currently has a size limit per event of 25MB, based on the [Log Analytics limit of 30MB per API Call](https://docs.microsoft.com/rest/api/loganalytics/create-request#data-limits) with additional overhead for formatting. The default behavior when hitting this limit is to throw an exception. This can be changed by modifying the value of `EXCEPTION_ON_FAILED_SEND` in [GenericSendBuffer.java](src/spark-listeners/src/main/java/com/microsoft/pnp/client/GenericSendBuffer.java) to `false`.

> Note: You will see an error like: `java.lang.RuntimeException: Failed to schedule batch because first message size nnn exceeds batch size limit 26214400 (bytes).` in the Spark logs if your workload is generating logging messages of greater than 25MB, and your workload may not proceed. You can query Log Analytics for this error condition with:

> ```kusto
> SparkLoggingEvent_CL
> | where TimeGenerated > ago(24h)
> | where Message contains "java.lang.RuntimeException: Failed to schedule batch because first message size"
> ```

## Build the Azure Databricks monitoring library

You can build the library using either Docker or Maven.  All commands are intended to be run from the base directory of the repository.

The jar files that will be produced are:

**`spark-monitoring_<Version>.jar`** - This is the implementation of the Spark Listener framework that provides capability for collecting data from the running cluster for forwarding to another logging system.

### Option 1: Docker

Linux:

```bash
# To build all profiles:
docker run -it --rm -v `pwd`:/spark-monitoring -v "$HOME/.m2":/root/.m2 mcr.microsoft.com/java/maven:8-zulu-debian10 /spark-monitoring/build.sh
```

```bash
# To build a single profile (example for latest long term support version 10.4 LTS):
docker run -it --rm -v `pwd`:/spark-monitoring -v "$HOME/.m2":/root/.m2 -w /spark-monitoring/src mcr.microsoft.com/java/maven:8-zulu-debian10 mvn install -P "scala-2.12_spark-3.3.1"
```

Windows:

```bash
# To build all profiles:
docker run -it --rm -v %cd%:/spark-monitoring -v "%USERPROFILE%/.m2":/root/.m2 mcr.microsoft.com/java/maven:8-zulu-debian10 /spark-monitoring/build.sh
```

```bash
# To build a single profile (example for latest long term support version 10.4 LTS):
docker run -it --rm -v %cd%:/spark-monitoring -v "%USERPROFILE%/.m2":/root/.m2 -w /spark-monitoring/src mcr.microsoft.com/java/maven:8-zulu-debian10 mvn install -P "scala-2.12_spark-3.3.1"
```

### Option 2: Maven

1. Import the Maven project object model file, _pom.xml_, located in the **/** folder into your project. This will import the project

1. Activate a **single** Maven profile that corresponds to the versions of the DBR version combination that is being used. By default, the DBR version 12.2LTS profile is active.

1. Execute the Maven **package** phase in your Java IDE to build the JAR files for the the project:

## Configure the Databricks workspace

Copy the JAR files and init scripts to Databricks.

1. Use the Azure Databricks CLI to create a workspace directory named **/databricks/spark-monitoring** for the init script and a DBFS directory named **dbfs:/databricks/spark-monitoring** for the JAR:

    ```bash
    databricks workspace mkdirs /databricks/spark-monitoring
    databricks fs mkdirs dbfs:/databricks/spark-monitoring
    ```

1. Open the **src/scripts/spark-monitoring.sh** script file and add your [Log Analytics Workspace ID and Key](http://docs.microsoft.com/azure/azure-monitor/platform/agent-windows#obtain-workspace-id-and-key) to the lines below:

    ```bash
    export LOG_ANALYTICS_WORKSPACE_ID=
    export LOG_ANALYTICS_WORKSPACE_KEY=
    ```

If you do not want to add your Log Analytics workspace id and key into the init script in plaintext, you can also [create an Azure Key Vault backed secret scope](./docs/keyvault-backed-secrets.md) and reference those secrets through your cluster's environment variables.

1. In order to add `x-ms-AzureResourceId` [header](https://docs.microsoft.com/azure/azure-monitor/platform/data-collector-api#request-headers) as part of the http request, modify the following environment
variables on **/src/scripts/spark-monitoring.sh**.
For instance:

```bash
export AZ_SUBSCRIPTION_ID=11111111-5c17-4032-ae54-fc33d56047c2
export AZ_RSRC_GRP_NAME=myAzResourceGroup
export AZ_RSRC_PROV_NAMESPACE=Microsoft.Databricks
export AZ_RSRC_TYPE=workspaces
export AZ_RSRC_NAME=myDatabricks
```

Now the _ResourceId **/subscriptions/11111111-5c17-4032-ae54-fc33d56047c2/resourceGroups/myAzResourceGroup/providers/Microsoft.Databricks/workspaces/myDatabricks** will be part of the header.
(*Note: If at least one of them is not set the header won't be included.*)

1. Use the Azure Databricks CLI to copy **src/scripts/spark-monitoring.sh** to the directory created in step 3:

    ```bash
    databricks workspace import /databricks/spark-monitoring/spark-monitoring.sh --format "AUTO" --content "$(base64 src/scripts/spark-monitoring.sh)" --overwrite
    ```

1. Use the Azure Databricks CLI to copy the jar file from the **target/** folder to the directory created in step 3:

    ```bash
    databricks fs cp --overwrite target/spark-monitoring_1.0.0.jar dbfs:/databricks/spark-monitoring/
    ```

### Create and configure the Azure Databricks cluster

1. Navigate to your Azure Databricks workspace in the Azure Portal.
1. Under "Compute", click "Create Cluster".
1. Choose a name for your cluster and enter it in "Cluster name" text box.
1. In the "Databricks Runtime Version" dropdown, select **Runtime: 11.3 LTS (Scala 2.12, Spark 3.3.1)**.
1. Under "Advanced Options", click on the "Spark" tab. Add env variables for LOG_ANALYTICS_WORKSPACE_KEY and LOG_ANALYTICS_WORKSPACE_KEY. Use this step if you haven't set those variables in the init script
1. Under "Advanced Options", click on the "Init Scripts" tab. Go to the last line under the "Init Scripts section" Under the "destination" dropdown, select "Workspace". Enter "/databricks/spark-monitoring/spark-monitoring.sh" in the text box. Click the "add" button.
1. Click the "Create Cluster" button to create the cluster. Next, click on the "start" button to start the cluster.

## Run the sample job (optional)

The repository includes a sample application that shows how to send application metrics and application logs to Azure Monitor.

When building the sample job, specify a maven profile compatible with your
databricks runtime from the [supported configurations section](#supported-configurations).

1. Use Maven to build the POM located at `sample/spark-sample-job/pom.xml` or run the following Docker command:

    Linux:

    ```bash
    docker run -it --rm -v `pwd`/sample/spark-sample-job:/spark-sample-job -v "$HOME/.m2":/root/.m2 -w /spark-sample-job mcr.microsoft.com/java/maven:8-zulu-debian10 mvn install -P <maven-profile>
    ```

    Windows:

    ```bash
    docker run -it --rm -v %cd%/sample/spark-sample-job:/spark-sample-job -v "%USERPROFILE%/.m2":/root/.m2 -w /spark-sample-job mcr.microsoft.com/java/maven:8-zulu-debian10 mvn install -P <maven-profile>
    ```

1. Navigate to your Databricks workspace and create a new job, as described [here](https://docs.microsoft.com/azure/databricks/workflows/jobs/jobs#--create-a-job).

1. In the job detail page, set **Type** to `JAR`.

1. For **Main class**, enter `com.microsoft.pnp.samplejob.StreamingQueryListenerSampleJob`.

1. Upload the JAR file from `/src/spark-jobs/target/spark-jobs-1.0-SNAPSHOT.jar` in the **Dependent Libraries** section.

1. Select the cluster you created previously in the **Cluster** section.

1. Select **Create**.

1. Click the **Run Now** button to launch the job.

When the job runs, you can view the application logs and metrics in your Log Analytics workspace. After you verify the metrics appear, stop the sample application job.

### Viewing the Sample Job's Logs in Log Analytics

After your sample job has run for a few minutes, you should be able to query for
these event types in Log Analytics:

#### SparkListenerEvent_CL

This custom log will contain Spark events that are serialized to JSON. You can limit the volume of events in this log with [filtering](docs/filtering.md#limiting-events-in-sparklistenerevent_cl). If filtering is not employed, this can be a large volume of data.

> Note: There is a known issue when the Spark framework or workload generates events that have more than 500 fields, or where data for an individual field is larger than 32kb. Log Analytics will generate an error indicating that data has been dropped. This is an incompatibility between the data being generated by Spark, and the current limitations of the Log Analytics API.

### Adding events in SparkListenerEvent_CL

You can add new events by adding new methods in the DatabricksListener class, the methods must override existing methods in the SparkListener interface.

Careful as some of them are extremely verbose and may not be sent in LogAnalytics. This is why you should test the events by running the SparkApp in the sample directory.


#### Example

Querying **SparkListenerEvent_CL** for job throughput over the last 7 days:

```kusto
let results=SparkListenerEvent_CL
| where TimeGenerated > ago(7d)
| where  Event_s == "SparkListenerJobStart"
| extend metricsns=column_ifexists("Properties_spark_metrics_namespace_s",Properties_spark_app_id_s)
| extend apptag=iif(isnotempty(metricsns),metricsns,Properties_spark_app_id_s)
| project Job_ID_d,apptag,Properties_spark_databricks_clusterUsageTags_clusterName_s,TimeGenerated
| order by TimeGenerated asc nulls last
| join kind= inner (
    SparkListenerEvent_CL
    | where Event_s == "SparkListenerJobEnd"
    | where Job_Result_Result_s == "JobSucceeded"
    | project Event_s,Job_ID_d,TimeGenerated
) on Job_ID_d;
results
| extend slice=strcat("#JobsCompleted ",Properties_spark_databricks_clusterUsageTags_clusterName_s,"-",apptag)
| summarize count() by bin(TimeGenerated, 1h),slice
| order by TimeGenerated asc nulls last
```

#### SparkLoggingEvent_CL

This custom log will contain data forwarded from Log4j (the standard logging system in Spark). The volume of logging can be controlled by [altering the level of logging](docs/filtering.md#limiting-logs-in-sparkloggingevent_cl-basic) to forward or with [filtering](docs/filtering.md#limiting-logs-in-sparkloggingevent_cl-advanced).

Example for querying **SparkLoggingEvent_CL** for logged errors over the last day:

```kusto
SparkLoggingEvent_CL
| where TimeGenerated > ago(1d)
| where log_level_s == "ERROR"
```

#### SparkMetric_CL

This custom log will contain metrics events as generated by the Spark framework or workload. You can adjust the time period or sources included by modifying [the `METRICS_PROPERTIES` section of the spark-monitoring.sh](src/spark-listeners/scripts/spark-monitoring.sh#L63-L76) script or by [enabling filtering](docs/filtering.md#limiting-metrics-in-sparkmetric_cl).

Example of querying **SparkMetric_CL** for the number of active executors per application over the last 7 days summarized every 15 minutes:

```kusto
SparkMetric_CL
| where TimeGenerated > ago(7d)
| extend sname=split(name_s, ".")
| where sname[2] == "executor"
| extend executor=strcat(sname[1]) 
| extend app=strcat(sname[0])
| summarize NumExecutors=dcount(executor) by bin(TimeGenerated,  15m),app
| order by TimeGenerated asc nulls last
```

> Note: For more details on how to use the saved search queries in [logAnalyticsDeploy.json](/perftools/deployment/loganalytics/logAnalyticsDeploy.json) to understand and troubleshoot performance, see [Observability patterns and metrics for performance tuning](https://docs.microsoft.com/azure/architecture/databricks-monitoring/databricks-observability).

## Filtering

The library is configurable to limit the volume of logs that are sent to each of the different Azure Monitor log types.  See [filtering](./docs/filtering.md) for more details.

## Debugging

If you encounter any issues with the init script, you can refer to the docs on [debugging](./docs/debugging.md).

## Contributing

See: [CONTRIBUTING.md](CONTRIBUTING.md)
