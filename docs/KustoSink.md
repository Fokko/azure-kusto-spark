# Kusto Sink Connector

Kusto sink connector allows writing data from Spark to a table 
in the specified Kusto cluster and database

## Authentication

Kusto connector uses **Azure Active Directory (AAD)** to authenticate the client application 
that is using it. Please verify the following before using Kusto connector:
 * Client application is registered in AAD
 * Client application has 'user' privileges or above on the target database
 * In addition, client application has 'ingestor' privileges on the target database
 * When writing to an existing table, client application has 'admin' privileges on the target table
 
 For details on Kusto principal roles, please refer to [Role-based Authorization](https://docs.microsoft.com/en-us/azure/kusto/management/access-control/role-based-authorization) 
 section in [Kusto Documentation](https://docs.microsoft.com/en-us/azure/kusto/).
 
 For managing security roles, please refer to [Security Roles Management](https://docs.microsoft.com/en-us/azure/kusto/management/security-roles) 
 section in [Kusto Documentation](https://docs.microsoft.com/en-us/azure/kusto/).
 
 ## Batch Sink: 'write' command
 
 Kusto connector implements Spark 'Datasource V1' API. 
 Kusto data source identifier is "com.microsoft.kusto.spark.datasource". 
 Dataframe schema is translated into kusto schema as explained in [DataTypes](Spark-Kusto%20DataTypes%20mapping.md).
 
 ### Command Syntax
 ```scala
 <dataframe-object>
 .write
 .format("com.microsoft.kusto.spark.datasource")
 .option(KustoSinkOptions.<option-name-1>, <option-value-1>
 ...
 .option(KustoSinkOptions.<option-name-n>, <option-value-n>
 .mode(SaveMode.Append)
 .save()
 ```
 
 ### Supported Options
 
 All the options that can be use in the Kusto sink are under the object KustoSinkOptions.
 
**Mandatory Parameters:** 
 
* **KUSTO_CLUSTER**:
 Target Kusto cluster to which the data will be written.
 Use either cluster profile name for global clusters, or <profile-name.region> for regional clusters.
 For example: if the cluster URL is 'https://testcluster.eastus.kusto.windows.net', set this property 
 as 'testcluster.eastus' 
  
 * **KUSTO_DATABASE**: 
 Target Kusto database to which the data will be written. The client must have 'user' and 'ingestor' 
 privileges on this database.
 
 * **KUSTO_TABLE**: 
 Target Kusto table to which the data will be written. If _KUSTO_CREATE_TABLE_OPTIONS_ is 
 set to "FailIfNotExist" (default), the table must already exist, and the client must have 
 'admin' privileges on the table.
 
 **Authentication Parameters** can be found here - [AAD Application Authentication](Authentication.md). 
 
 **Optional Parameters:** 
 * **KUSTO_TABLE_CREATE_OPTIONS**: 
 If set to 'FailIfNotExist' (default), the operation will fail if the table is not found 
 in the requested cluster and database.  
 If set to 'CreateIfNotExist' and the table is not found in the requested cluster and database,
 it will be created, with a schema matching the DataFrame that is being written.
 
 * **KUSTO_WRITE_ENABLE_ASYNC**:
  If set to 'false' (default), writing to Kusto is done synchronously. This means:
   * Once the operation completes successfully, it is guaranteed that that data was written to
 the requested table in Kusto
   * Exceptions will propagate to the client
 
   This is the recommended option for typical use cases. However, using it results in blocking
 Spark driver for as long as the operation is running, up to several minutes. 
 To **avoid blocking Spark driver**, it is possible to execute Kusto 'write' operation in an 
 opportunistic mode as an asynchronous operation, this is recommended only for spark streaming.
  The resulted behavior is the following:
   * Spark driver is not blocked
   * In a success scenario, all data will eventually be written, only if the job is left running.  Job success status **DOES NOT** mean data is committed yet.
   * In a failure scenario, error messages are logged on Spark executor nodes, 
 but exceptions will not propagate to the client
 
 * **KUSTO_TIMEOUT_LIMIT**:
   An integer number corresponding to the period in seconds after which the operation will timeout.
   This is an upper limit that may coexist with addition timeout limits as configured on Spark or Kusto clusters.  
   Default: '5400' (90 minutes)

* **KustoSinkOptions.KUSTO_SPARK_INGESTION_PROPERTIES_JSON**:
    A json representation of a `SparkIngestionProperties` (use `toString` to make a json of an instance).
    
    Properties:
        
    - dropByTags, ingestByTags, additionalTags, ingestIfNotExists: util.ArrayList[String] - 
    Tags list to add to the extents. Read [kusto docs - extents](https://docs.microsoft.com/en-us/azure/kusto/management/extents-overview#ingest-by-extent-tags)
    
    - creationTime: DateTime - sets the extents creationTime value to this date
    
    - csvMapping: String - a full json representation of a csvMapping (the connector always upload csv files to Kusto), 
    see here [kusto docs - mappings](https://docs.microsoft.com/en-us/azure/kusto/management/mappings)
    
    - csvMappingNameReference: String - a reference to a name of a csvMapping pre-created for the table  
    
    - flushImmediately: Boolean - use with caution - flushes the data immidiatly upon ingestion without aggregation.

* **KUSTO_CLIENT_BATCHING_LIMIT**:
    A limit indicating the size in MB of the aggregated data before ingested to Kusto. Note that this is done for each
    partition. The ingestion Kusto also aggregates data, default suggested by Kusto is 1GB but here we suggest to cut 
    it at 100MB to adjust it to spark pulling of data.
    
 >**Note:**
 For both synchronous and asynchronous operation, 'write' is an atomic transaction, i.e. 
 either all data is written to Kusto, or no data is written. 
 
### Performance Considerations

Write performance depends on multiple factors, such as cluster scale of both Spark and Kusto clusters.
WIth regards to Kusto target cluster configuration, one of the factors that impacts performance and latency 
is [Ingestion Batching Policy](https://docs.microsoft.com/en-us/azure/kusto/concepts/batchingpolicy). Default policy 
works well for typical scenarios, especially when writing large amounts of data as batch. For reduced latency,
consider altering the policy to a relatively low value (minimal allowed is 10 seconds).
**This is mostly relevant when writing to Kusto in streaming mode**.
For more details and command reference, please see [Ingestion Batching Policy command reference](https://docs.microsoft.com/en-us/azure/kusto/management/batching-policy).
 
### Examples

Synchronous mode, table already exists:
```scala
df.write
  .format("com.microsoft.kusto.spark.datasource")
  .option(KustoSinkOptions.KUSTO_CLUSTER, "MyCluster.RegionName")
  .option(KustoSinkOptions.KUSTO_DATABASE, "MyDatabase")
  .option(KustoSinkOptions.KUSTO_TABLE, "MyTable")
  .option(KustoSinkOptions.KUSTO_AAD_CLIENT_ID, "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
  .option(KustoSinkOptions.KUSTO_AAD_CLIENT_PASSWORD, "MyPassword") 
  .option(KustoSinkOptions.KUSTO_AAD_AUTHORITY_ID, "AAD Authority Id") // "microsoft.com"
  .mode(SaveMode.Append)
  .save()
``` 

IngestionProperties and short scala usage:
```scala
val sp = new SparkIngestionProperties
var tags = new util.ArrayList[String]()
tags.add("newTag")
sp.ingestByTags = tags
sp.creationTime = new DateTime().minusDays(1)
df.write.kusto(cluster,
             database,
             table, 
             conf, // optional
             Some(sp)) // optional
```

Asynchronous mode, table may not exist and will be created:
```scala
df.write
  .format("com.microsoft.kusto.spark.datasource")
  .option(KustoOptions.KUSTO_CLUSTER, "MyCluster.RegionName")
  .option(KustoOptions.KUSTO_DATABASE, "MyDatabase")
  .option(KustoOptions.KUSTO_TABLE, "MyTable")
  .option(KustoOptions.KUSTO_AAD_CLIENT_ID, "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
  .option(KustoOptions.KUSTO_AAD_CLIENT_PASSWORD, "MyPassword") 
  .option(KustoOptions.KUSTO_AAD_AUTHORITY_ID, "AAD Authority Id") // "microsoft.com"
  .option(KustoOptions.KUSTO_WRITE_ENABLE_ASYNC, true)
  .option(KustoOptions.KUSTO_TABLE_CREATE_OPTIONS, "CreateIfNotExist")
  .mode(SaveMode.Append)
  .save()
```

 ## Streaming Sink: 'writeStream' command
 
 Kusto sink connector was adapted to support writing from a streaming source to a Kusto table.
 
 ### Command Syntax
  ```scala
  <queue-name> = <streaming-source-name>
   .writeStream
   .format("com.microsoft.kusto.spark.datasink.KustoSinkProvider")
   .options(Map(
      KustoOptions.<option-name-1>, <option-value-1>,
      ...,
      KustoOptions.<option-name-n>, <option-value-n>))
   .trigger(Trigger.Once)
  ```
 ### Example
 ```scala
 var customSchema = new StructType().add("colA", StringType, nullable = true).add("colB", IntegerType, nullable = true)
 
 // Read data to stream 
 val csvDf = spark
       .readStream      
       .schema(customSchema)
       .csv("/FileStore/tables")
 
 spark.conf.set("spark.sql.streaming.checkpointLocation", "/FileStore/temp/checkpoint")
 spark.conf.set("spark.sql.codegen.wholeStage", "false")
 
 val kustoQ = csvDf
       .writeStream
       .format("com.microsoft.kusto.spark.datasink.KustoSinkProvider")
       .options(Map(
         KustoOptions.KUSTO_CLUSTER -> cluster,
         KustoOptions.KUSTO_TABLE -> table,
         KustoOptions.KUSTO_DATABASE -> database,
         KustoOptions.KUSTO_AAD_CLIENT_ID -> appId,
         KustoOptions.KUSTO_AAD_CLIENT_PASSWORD -> appKey,
         KustoOptions.KUSTO_AAD_AUTHORITY_ID -> authorityId))
       .trigger(Trigger.Once)
 
 kustoQ.start().awaitTermination(TimeUnit.MINUTES.toMillis(8))      
 ```
 
  For more reference code examples please see: 
  
   [SimpleKustoDataSink](../samples/src/main/scala/SimpleKustoDataSink.scala)
   
   [KustoConnectorDemo](../samples/src/main/scala/KustoConnectorDemo.scala)
   
   [Python samples](../samples/src/main/python/pyKusto.py)
