# Migrate Cassandra data with Azure Databricks

This sample allows you to migrate data between tables in Apache Cassandra using Spark with Azure Databricks, while preserving the original `writetime`. This can be useful when doing historic data loads during a [live migration](https://docs.microsoft.com/azure/managed-instance-apache-cassandra/dual-write-proxy-migration).

# Setup Azure Databricks

## Prerequisites

* [Provision an Azure Databricks cluster](https://docs.microsoft.com/azure/databricks/scenarios/quickstart-create-databricks-workspace-portal?tabs=azure-portal). Ensure it also has network access to your source and target Cassandra clusters.

* Ensure you've already migrated the keyspace/table schema from your source Cassandra database to your target Cassandra database.


## Provision a Spark cluster

We recommend selecting Azure Databricks runtime version 7.5, which supports Spark 3.0.

<!-- :::image type="content" source="https://docs.microsoft.com/azure/cosmos-db/media/cassandra-migrate-cosmos-db-databricks/databricks-runtime.png" alt-text="Screenshot that shows finding the Databricks runtime version."::: -->
![Databricks runtime](https://docs.microsoft.com/azure/cosmos-db/media/cassandra-migrate-cosmos-db-databricks/databricks-runtime.png)

## Add Cassandra Migrator Spark dependencies

* Download the dependency jar [here](https://github.com/Azure-Samples/cassandra-migrator/raw/main/jar/cassandra-migrator-assembly-0.0.1.jar) * 
* Upload and install the jar on your Databricks cluster:

<!-- :::image type="content" source="./media/cassandra-migrator-jar.jpg" alt-text="Screenshot that shows searching for Maven packages in Databricks."::: -->
![Dependency jar](./media/cassandra-migrator-jar.jpg)

Select **Install**, and then restart the cluster when installation is complete.

\* You can also build the dependency jar using [SBT](https://www.scala-sbt.org/1.x/docs/Setup.html) by running `build.sh` in the /build_files directory of this repo.

> [!NOTE]
> Make sure that you restart the Databricks cluster after the dependency jar has been installed.

# Migrate Cassandra tables

Create a new Scala notebook in Databricks with two seperate cells:

### Read Cassandra source table

In this case, we are migrating from a source cluster which does not implement SSL, to a target table which does. You can adjust `sslOptions` for your source/target tables accordingly.

```scala
import org.apache.spark.sql._

val spark = SparkSession
      .builder()
      .appName("cassandra-migrator")
      .config("spark.task.maxFailures", "1024")
      .config("spark.stage.maxConsecutiveAttempts", "60") 
      .getOrCreate

import com.cassandra.migrator.readers.Cassandra
import com.cassandra.migrator.config._
import com.datastax.spark.connector.cql.CassandraConnector;

val cassandraSource = new SourceSettings.Cassandra(
  host = "<source Cassandra host name/IP here>",
  port = 9042,
  localDC = Some(null),
  credentials = Some(Credentials(
    username="<username here>", 
    password="<password here>")
  ),
  sslOptions = Some(SSLOptions(
    clientAuthEnabled=false,
    enabled=false,
    trustStorePassword = Some(null),
    trustStorePath = Some(null),
    trustStoreType = Some(null),
    keyStorePassword = Some(null),
    keyStorePath = Some(null),
    keyStoreType = Some(null),
    enabledAlgorithms = Some(null),
    protocol = Some("TLS")
  )),  
  keyspace = "<source keyspace name>",
  table = "<source table name>",
  splitCount = Some(1), // Number of splits to use - this should be at minimum the amount of cores available in the Spark cluster, and optimally more; higher splits will lead to more fine-grained resumes. Aim for 8 * (Spark cores).
  connections = Some(1), // Number of connections to use to Cassandra when copying
  fetchSize = 1000, // Number of rows to fetch in each read
  preserveTimestamps = true, // Preserve TTLs and WRITETIMEs of cells in the source database. Note that this option is *incompatible* when copying tables with collections (lists, maps, sets).
  where = None // Optional condition to filter source table data that will be migrated, e.g. where: race_start_date = '2015-05-27' AND race_end_date = '2015-05-27'
)

val sourceDF = Cassandra.readDataframe(
  spark,
  cassandraSource,
  cassandraSource.preserveTimestamps,
  tokenRangesToSkip = Set()
)
sourceDF.dataFrame.printSchema()
```

### Migrate to Cassandra target table

```scala
import com.cassandra.migrator.writers

implicit val spark = SparkSession
      .builder()
      .appName("cassandra-migrator")
      .config("spark.task.maxFailures", "1024")
      .config("spark.stage.maxConsecutiveAttempts", "60")
      .getOrCreate

val target = new TargetSettings.Cassandra(
  host = "<target Cassandra host name/IP>",
  port = 9042,
  localDC = Some(null),
  credentials = Some(com.cassandra.migrator.config.Credentials(
    username="<username here>", 
    password="<password here>")
  ),
  sslOptions = Some(SSLOptions(
    clientAuthEnabled=false,
    enabled=true,
    trustStorePassword = Some(null),
    trustStorePath = Some(null),
    trustStoreType = Some(null),
    keyStorePassword = Some(null),
    keyStorePath = Some(null),
    keyStoreType = Some(null),
    enabledAlgorithms = Some(Set("TLS_RSA_WITH_AES_128_CBC_SHA","TLS_RSA_WITH_AES_256_CBC_SHA")),
    protocol = Some("TLS")
  )),   
  keyspace = "<target keyspace name>",
  table = "<target table name>",
   connections = Some(1),
  stripTrailingZerosForDecimals = false
)

writers.Cassandra.writeDataframe(
            target,
            List(),
            sourceDF.dataFrame,
            sourceDF.timestampColumns
)
```