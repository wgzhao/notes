# Integrating Apache Hive with Apache Spark - HWAR

[Source](https://community.hortonworks.com/content/kbentry/223626/integrating-apache-hive-with-apache-spark-hive-war.html "Permalink to Integrating Apache Hive with Apache Spark - Hive Warehouse Connector - Hortonworks")

#### Article

**Short Description**: This article targets to describe and demonstrate Apache Hive Warehouse Connector which is a newer generation to read and write data between Apache Spark and Apache Hive.

## 1\. Motivation

Apache Spark and Apache Hive integration has always been an important use case and continues to be so. Both provide their own efficient ways to process data by the use of SQL, and is used for data stored in distributed file systems. Both provide compatibilities for each other. As both systems evolve, it is critical to find a solution that provides the best of both worlds for data processing needs.

 In case of Apache Spark, it provides [a basic Hive compatibility](https://spark.apache.org/docs/latest/sql-programming-guide.html#compatibility-with-apache-hive). It allows an access to tables in Apache Hive and some basic use cases can be achieved by this. However, not all the modern features from Apache Hive are supported, for instance, [ACID](https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions) table in Apache Hive, [Ranger integration](https://community.hortonworks.com/articles/148917/orc-improvements-for-apache-spark-22.html), [Live Long And Process (LLAP)](https://cwiki.apache.org/confluence/display/Hive/LLAP), etc. as of Spark 2.

 Apache Spark supports a pluggable approach for various data sources and Apache Hive itself can be also considered as one data source. Therefore, this library, Hive Warehouse Connector, was implemented as a data source to overcome the limitations and provide those modern functionalities in Apache Hive to Apache Spark users.

**Note**: From HDP 3.0, catalogs for Apache Hive and Apache Spark are separated, and they use their own catalog; namely, they are mutually exclusive - Apache Hive catalog can only be accessed by Apache Hive or this library, and Apache Spark catalog can only be accessed by existing APIs in Apache Spark . In other words, some features such as ACID tables or Apache Ranger with Apache Hive table are only available via this library in Apache Spark. Those tables in Hive should not directly be accessible within Apache Spark APIs themselves.

## 2\. Introduction

 This library provides both Scala (Java compatible) and Python APIs for:

* SQL / DataFrame APIs interacting with both transactional and non-transactional tables in Apache Hive
* SQL / DataFrame read support
* SQL / DataFrame and Structured Streaming write support

### 2.1. SQL / DataFrame Read Support

* [Live Long And Process (LLAP)](https://cwiki.apache.org/confluence/display/Hive/LLAP) is fully utilized which Apache Hive introduced for faster performance
* Apache Spark's [Apache Arrow integration](https://issues.apache.org/jira/browse/SPARK-21187) is fully utilized for vectorized operations, faster and compact data interactions
* It is implemented by [Data Source V2](https://issues.apache.org/jira/browse/SPARK-15689) which has a columnar format support and various functionalities

 ![](https://lh4.googleusercontent.com/-SNAWFl3QqG0xtYidk6ByT_hi6r7ypH4Z8OyFL4592yDfD6TU6_0kfFxmoKfyb_Sb5NT3x2zvSOVlYW3cvdUCaVnJ2xICMhUKa7ohzY0-KszX2H-Qjr6UvuIMxGyMpbs2q3S_uAT)

**Note:** This diagram shows read execution path to illustrate the key points.

 For instance, it is able to read an Apache Hive table in Apache Spark as below:

    import com.hortonworks.hwc.HiveWarehouseSession
    val hive = HiveWarehouseSession.session(spark).build()
    val df = hive.executeQuery("SELECT * FROM tableA")

 It leverages Apache Hive LLAP and retrieves data from Hive table into Spark DataFrame.

### 2.2. SQL / DataFrame & Structured Streaming Write Support

* [Hive Streaming API](https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest+V2) is used in both batch and streaming write, which Apache Hive introduced to continuously digest data.
* Since it is implemented by [Data Source V2](https://issues.apache.org/jira/browse/SPARK-15689), it supports a commit protocol and supports atomic write operation.
* Native Apache ORC writer is used instead, which has many important fixes in terms of performance and stability.

 The code below illustrate writing data from Apache Spark to Apache Hive table in Structured Streaming.

    import com.hortonworks.hwc.HiveWarehouseSession
    val hive = HiveWarehouseSession.session(spark).build()
    df.writeStream
      .format(HiveWarehouseSession.STREAM_TO_STREAM)
      .option("table", "hwx_table").start()

Internally this fully utilizes Apache Hive’s Streaming API to write Apache’s DataFrame out to Apache Hive’s table.

** Note**: This does not need Hive LLAP daemons to be running.

## 3\. Prerequisites

 In Apache Hive:

* ‘Interactive Query' (LLAP) should be enabled (see **7\. Appendix** for Apache Ambari UI)

 In Apache Spark:

* `spark.hadoop.hive.llap.daemon.service.hosts` should be set for the application name of the LLAP service since this library utilizes LLAP. For example, `@llap0`.
* HiveServer2's JDBC URL should be specified in `spark.sql.hive.hiveserver2.jdbc.url` as well as configured in the cluster. For example, `jdbc:hive2://localhost:10000`. Use the HiveServer2 Interactive JDBC URL, rather than the traditional HiveServer2's JDBC URL.
* Make sure `spark.datasource.hive.warehouse.load.staging.dir` is pointed into a suitable HDFS-compatible staging directory, e.g. `/tmp`.
* Also, ensure `spark.datasource.hive.warehouse.metastoreUri` is configured properly. For example, `thrift://localhost:9083` to indicate Metastore URI.
* Note that `spark.security.credentials.hiveserver2.enabled` should be set to `false` for YARN client deploy mode, and `true` for YARN cluster deploy mode (by default). This configuration is required for a Kerberized cluster.
* When `spark.security.credentials.hiveserver2.enabled` is set to `false`, `spark.sql.hive.hiveserver2.jdbc.url.principal` can be optionally set if `spark.sql.hive.hiveserver2.jdbc.url` does not contain `principal`, for example, `hive/_HOST@EXAMPLE.COM`.
* When `spark.security.credentials.hiveserver2.enabled` is set to `true`, `spark.sql.hive.hiveserver2.jdbc.url.principal` should be set, for example, `hive/_HOST@EXAMPLE.COM`. Also, `spark.sql.hive.hiveserver2.jdbc.url` should not contain `principal`.

## 4\. User Scenarios

In this chapter, HDP 3.0.1.0 with Spark2, Hive and Ranger on a Kerberized cluster is used. Note that If you are running the examples on Kerberized cluster, for instance in HDP, make sure to obtain (or renew) the Kerberos ticket-granting ticket by kinit with an appropriate keytab.

The following scenarios are run by Spark shell as below. For other interaction paradigms, like spark-submit, Livy, Zeppelin etc. please see the subsequent sections for specific setup steps.

 In case of Scala:

    $ spark-shell --master yarn \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --conf spark.security.credentials.hiveserver2.enabled=false

 In case of Python:

    $ pyspark --master yarn \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --py-files /usr/hdp/3.0.1.0-183/hive_warehouse_connector/pyspark_hwc-1.0.0.3.0.1.0-183.zip \
      --conf spark.security.credentials.hiveserver2.enabled=false

### 4.1. SQL / DataFrame Read

 This scenario targets to demonstrate a bulk read operation, as a batch job, from Hive table to Spark DataFrame with SQL expression. For this scenario, data from FBI crime rate is used (see **7\. Appendix** for `SQL INSERT` queries).

 Before we start, we should initialize `HiveWarehouseSession` instance.

 In case of Scala:

    import com.hortonworks.hwc.HiveWarehouseSession
    val hive = HiveWarehouseSession.session(spark).build()

 In case of Python:

    from pyspark_llap.sql.session import HiveWarehouseSession
    hive = HiveWarehouseSession.session(spark).build()

 In this scenarios, it uses hwc\_db database. The code below shows the decreasing crime rate (per 100,000 Inhabitants) between 2000 and 2010 in USA:

    hive.setDatabase("hwc_db")
    hive.table("crimes").filter("year = 2000 OR year = 2010").show()
    +----+----------+
    |year|crime_rate|
    +----+----------+
    |2010|     404.5|
    |2000|     506.5|
    +----+----------+

 SQL queries can be also directly executed via executeQuery API, which directly interacts with Hive LLAP daemons.

    hive.executeQuery("SELECT avg(crime_rate) AS average_crime_rate FROM crimes").show()
    +------------------+
    |average_crime_rate|
    +------------------+
    |456.33500000000004|
    +------------------+

### 4.2. SQL / DataFrame Read (Apache Ranger Integration)

As of HDP 2.6, Row/Column Level Access Control for Apache Spark is available that enables fine grained security control leveraged by Apache Ranger. This feature can also be leveraged by this library - Apache Spark APIs do not support this.

For instance, the scenarios below demonstrate row level control with users, `billing`, and `datascience`. `billing` principal can access all rows and columns while `datascience` principal can access some of filtered and masked data.

Firstly, we destroy existing ticket and obtains a ticket as `billing` principal. Then, it executes a Spark shell.

    $ kdestroy
    $ kinit billing/billing@EXAMPLE.COM
    $ spark-shell --master yarn \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --conf spark.security.credentials.hiveserver2.enabled=false

Since `billing` principal can access every row, it shows all data as are.

    sql("select * from db_spark.t_spark").show()
    +------+-------+
    |fruits|  color|
    +------+-------+
    | apple|    red|
    | grape| purple|
    |orange|scarlet|
    +------+-------+

Now, we try the same query as `datascience`, principal.

    $ kdestroy
    $ kinit datascience/datascience@EXAMPLE.COM
    $ spark-shell --master yarn \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --conf spark.security.credentials.hiveserver2.enabled=false

Since `datascience`, principal can only access to filtered and masked rows, it only shows partial values.

    sql("select * from db_spark.t_spark").show()
    +------+-----+
    |fruits|color|
    +------+-----+
    | apxxx|  red|
    +------+-----+

For more information, please see 'Introducing Row/ Column Level Access Control for Apache Spark' in **7\. Appendix**. The Ranger security features previously available in the spark-llap connector are now covered by the Hive Warehouse Connector. Apache Hive and Apache Spark configurations should be followed as described in **3\. Prerequisites**.

### 4.3. SQL / DataFrame Write

 This scenario targets to demonstrate a bulk write operation, as a batch job, between Apache Hive table and Apache Spark DataFrame with SQL expression.

 A table in Hive can be created as below:

    hive.createTable("crimes_2010")
      .column("year", "int")
      .column("crime_rate", "double")
      .create()

 Here, crimes table (from **4.1 SQL / DataFrame Read**) is written into a different Hive table after filtering the data in Spark. The code below writes the crime rate at 2010 into the table created above:

    hive.table("crimes").filter("year = 2010")
      .write
      .format(HiveWarehouseSession.HIVE_WAREHOUSE_CONNECTOR)
      .option("table", "crimes_2010")
      .save()

 The data can also be written in a stream manner as below:

    hive.table("crimes").filter("year = 2010")
      .write
      .format(HiveWarehouseSession.DATAFRAME_TO_STREAM)
      .option("table", "crimes_2010")
      .save()

 **Note**: In this API, the job at Apache Spark side is a batch write job but it writes out data internally by Apache Hive Streaming API.

 Of course, we can write data from Apache Hive table via Apache Spark data sources:

    hive.table("crimes")
      .write
      .format("csv")
      .option("header", "true")
      .save("/tmp/zoo")

 Likewise, Apache Spark DataFrame can directly write out to Apache Hive table as well:

    spark.read.format("csv").option("inferSchema", "true").option("header", "true").load("/tmp/zoo")
      .filter("year = 2010")
      .write
      .format(HiveWarehouseSession.HIVE_WAREHOUSE_CONNECTOR)
      .option("table", "crimes_2010")
      .save()

### 4.4. Structured Streaming Write

 This scenario demonstrates a streaming write operation, as a micro batch job, from Apache Spark DataFrame to Apache Hive table with SQL expression.

 In this scenario, socket structured streaming is used as test data.

 In case of Scala:

    val lines = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 9999)  .load()

 In case of Python:

    lines = spark.readStream \
      .format("socket") \
      .option("host", "localhost") \
      .option("port", 9999) \
      .load()

 In another terminal, execute below command to insert test data:

    $ nc -lk 9999

 We create a target table to store the stream data. Note that this table can be created in Hive side as well, for instance, `CREATE TABLE hwx_table(value STRING);`

    hive.createTable("hwx_table").column("value", "string").create()

 Now, the stream data is ready. After that, import the library so that we can easily specify the format.

 Before we start, we should initialize `HiveWarehouseSession` instance.

 In case of Scala:

    import com.hortonworks.hwc.HiveWarehouseSession

 In case of Python:

    from pyspark_llap.sql.session import HiveWarehouseSession

 Next, it starts the structured streaming job. At the terminal which opened `nc -lk 9999` we can insert arbitrary data and when the terminal inserts Hortonworks, it saves the data into the `hwx_table` table.

    lines.filter("value = 'Hortonworks'")
      .writeStream
      .format(HiveWarehouseSession.STREAM_TO_STREAM)
      .option("database", "hwc_db")
      .option("table", "hwx_table")
      .option(
        "metastoreUri",
        spark.conf.get("spark.datasource.hive.warehouse.metastoreUri"))
      .option("checkpointLocation", "/tmp/checkpoint").start()

 **Note**: There is an issue about interpreting `spark.datasource.*` configurations into options internally in Apache Spark, which currently makes this library require to set `metastoreUri` and `database` options manually. See [SPARK-25460](https://issues.apache.org/jira/browse/SPARK-25460) for more details. As soon as this issue is resolved, both `metastoreUri` and `database` can be omitted likewise.

 After that, type Hortonworks and also arbitrary words in the terminal opening `nc -lk 9999`

    Foo
    Bar
    Hortonworks

 This continuously inserts the word Hortonworks into the table `hwx_table` .

    hive.table("hwx_table").show()
    +------------+
    |       value|
    +------------+
    | Hortonworks|
    +------------+

## 5\. Interaction Paradigms

 This library can be interacted with, of course, Apache Spark but also Apache Zeppelin and Apache Livy. This chapter demonstrates entry points of those interaction paradigms in order to leverage this library to utilize provided functionalities.

### 5.1. Spark Shell:

    $ spark-shell --master yarn \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --conf spark.security.credentials.hiveserver2.enabled=false

### 5.2. PySpark Shell:

    $ pyspark --master yarn \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --py-files /usr/hdp/3.0.1.0-183/hive_warehouse_connector/pyspark_hwc-1.0.0.3.0.1.0-183.zip \
      --conf spark.security.credentials.hiveserver2.enabled=false

### 5.3. Scala / Java Application Submission:

** In case of client mode:**

    $ spark-submit --master yarn --deploy-mode client \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --conf spark.security.credentials.hiveserver2.enabled=false [JAR_NAME]

If **3\. Prerequisites** at Apache Spark side was not globally configured, use the command with specifying all configurations, for instance:

    $ spark-submit --master yarn --deploy-mode client \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --conf spark.hadoop.hive.llap.daemon.service.hosts=@llap0 \
      --conf spark.sql.hive.hiveserver2.jdbc.url="jdbc:hive2://ctr-e138-1518143905142-509736-01-000007.hwx.site:2181,ctr-e138-1518143905142-509736-01-000009.hwx.site:2181,ctr-e138-1518143905142-509736-01-000006.hwx.site:2181,ctr-e138-1518143905142-509736-01-000005.hwx.site:2181,ctr-e138-1518143905142-509736-01-000008.hwx.site:2181/default;principal=hive/_HOST@HWX.COM;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2-interactive" \
      --conf spark.security.credentials.hiveserver2.enabled=false \
      --conf spark.datasource.hive.warehouse.load.staging.dir=/tmp \
      --conf spark.datasource.hive.warehouse.metastoreUri=thrift://ctr-e138-1518143905142-509736-01-000003.hwx.site:9083,thrift://ctr-e138-1518143905142-509736-01-000004.hwx.site:9083 [JAR_NAME]

 **In case of cluster mode:**

    $ spark-submit --master yarn --deploy-mode cluster \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar [JAR_NAME]

If **3\. Prerequisites** at Apache Spark side was not globally configured with HWC, use the command with specifying all configurations, for instance:

    $ spark-submit --master yarn --deploy-mode cluster \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --conf spark.hadoop.hive.llap.daemon.service.hosts=@llap0 \
      --conf spark.sql.hive.hiveserver2.jdbc.url="jdbc:hive2://ctr-e138-1518143905142-509736-01-000007.hwx.site:2181,ctr-e138-1518143905142-509736-01-000009.hwx.site:2181,ctr-e138-1518143905142-509736-01-000006.hwx.site:2181,ctr-e138-1518143905142-509736-01-000005.hwx.site:2181,ctr-e138-1518143905142-509736-01-000008.hwx.site:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2-interactive" \
      --conf spark.sql.hive.hiveserver2.jdbc.url.principal="hive/_HOST@EXAMPLE.COM" \
      --conf spark.datasource.hive.warehouse.load.staging.dir=/tmp \
      --conf spark.datasource.hive.warehouse.metastoreUri=thrift://ctr-e138-1518143905142-509736-01-000003.hwx.site:9083,thrift://ctr-e138-1518143905142-509736-01-000004.hwx.site:9083 [JAR_NAME]

### 5.4. Python Application Submission:

** In case of client mode:**

    $ spark-submit --master yarn --deploy-mode client \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --py-files /usr/hdp/3.0.1.0-183/hive_warehouse_connector/pyspark_hwc-1.0.0.3.0.1.0-183.zip \
      --conf spark.security.credentials.hiveserver2.enabled=false [PY_FILE]

** In case of cluster mode:**

    $ spark-submit --master yarn --deploy-mode cluster \
      --jars /usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar \
      --py-files /usr/hdp/3.0.1.0-183/hive_warehouse_connector/pyspark_hwc-1.0.0.3.0.1.0-183.zip [PY_FILE]

### 5.5. Apache Zeppelin

 Go to [the interpreter setting](https://zeppelin.apache.org/docs/0.8.0/usage/interpreter/overview.html).

 ![](https://lh6.googleusercontent.com/-7HxlmdUWRT8A94q7MBV3olT5nFpcOqcEJfAZOB9Ldt-HK8bnFqlbJMaGKZuOVKrWLa0-wZAPtYvOCPDmvcglMhMABSBb36CtDqXBUAcaSlKodteuqE96sKOyJDS402JF80kImY3)

 Find the interpreter settings and set the configuration accordingly.

 ** In case of Spark interpreter:**

 ![](https://lh6.googleusercontent.com/nlRa4H7_CObc_tUGs0ggYRjYP7APLo6BB-KTNnqyYXwvTnxWNQYgE5O0MBdZaAJoUBKmhbCvzGLYR0l8VxYfL8FS5mOW59CsOgJceYCxBthmxrrm0xf-RXWJH0Obr3fZrpfIwur0)

 Add `spark.jars` for instance:

    spark.jars=/usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar

 For Python usage, the configuration below should also be added. For instance:

    spark.submit.pyFiles=/usr/hdp/3.0.1.0-183/hive_warehouse_connector/pyspark_hwc-1.0.0.3.0.1.0-183.zip

 If the interpreter is set to YARN client mode, `spark.security.credentials.hiveserver2.enabled` should be set to false as below:

    spark.security.credentials.hiveserver2.enabled=false

 If you use a Kerberized cluster, do not forget to set:

    spark.yarn.keytab
    spark.yarn.principal

 ** In case of Livy interpreter:**

![](https://lh3.googleusercontent.com/3KpNt4bBdKy04XcyV8nGKiIHIsxEWTUZci8vBi2hE5j0ykoNoaN_RikgXbIuA8qqP7oMIUveTDWI6wn8MWaaSvLhQKVXJZUxdE4rwIzCLjkTqhqOxt20yBBFpREcnzdVIRbB9Idn)

 Add `livy.spark.jars` for instance:

    livy.spark.jars=file:/usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar

 For Python usage, the configuration below should also be added. For instance:

    livy.spark.submit.pyFiles=file:/usr/hdp/3.0.1.0-183/hive_warehouse_connector/pyspark_hwc-1.0.0.3.0.1.0-183.zip

 **Note**: Local directories are disallowed by default for security reasons. Set the local directory to `livy.file.local-dir-whitelist` to allow a specific directory to allow to read. To avoid this setting, please use HDFS path after uploading the JAR and zipped file.

 If the interpreter is set to YARN client mode (see `livy.spark.master` and `livy.spark.submit.deployMode`), `livy.spark.security.credentials.hiveserver2.enabled` should be set to `false` as below:

    livy.spark.security.credentials.hiveserver2.enabled=false

 If you use a Kerberized cluster, do not forget to set:

    zeppelin.livy.keytab
    zeppelin.livy.principal

**Note**: To use HWC with Livy in a secure cluster please follow the documentation [here](https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/using-zeppelin/content/using_spark_hwc_and_shc_client_jar_files_with_livy.html).

### 5.6. Apache Livy

The code below describes an example to submit Python application by Livy request with `curl` .

    $ curl -X POST -H "Content-Type: application/json" -H "X-Requested-By: hive" --negotiate  \
      -u : `hostname`:8999/batches --data '{
      "conf": {
        "spark.master": "yarn-cluster",
        "spark.jars": "file:/usr/hdp/3.0.1.0-183/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.1.0-183.jar", 
        "spark.submit.pyFiles": "file:/usr/hdp/3.0.1.0-183/hive_warehouse_connector/pyspark_hwc-1.0.0.3.0.1.0-183.zip"
      }, 
      "file": "[PY_FILE]"}' | python -m json.tool

It shows the job information.

    {
      "appId": null,
      "appInfo": {
        "driverLogUrl": null,
        "sparkUiUrl": null
      },
      "id": 1,
      "log": [
        "stdout: ",
        "\nstderr: ",
        "\nYARN Diagnostics: "
      ],
      "state": "starting"
    }

To patch job information, GET `batches/[ID]` is used.

    $ curl --negotiate -u : `hostname`:8999/batches/1 | python -m json.tool

This request shows an output, for example, as below. 

    {
      "appId": "application_1539107884019_0071",
      "appInfo": {
        "driverLogUrl": "http://ctr-e138-1518143905142-512381-01-000003.hwx.site:8188/applicationhistory/logs/ctr-e138-1518143905142-512381-01-000006.hwx.site:25454/container_e02_1539107884019_0071_01_000001/container_e02_1539107884019_0071_01_000001/hive",
        "sparkUiUrl": "http://ctr-e138-1518143905142-512381-01-000004.hwx.site:8088/proxy/application_1539107884019_0071/"
      },
      "id": 1,
      "log": [
        "\t queue: default",
        "\t start time: 1539151855183",
        "\t final status: UNDEFINED",
        "\t tracking URL: http://ctr-e138-1518143905142-512381-01-000004.hwx.site:8088/proxy/application_1539107884019_0071/",
        "\t user: hive",
        "18/10/10 06:10:55 INFO ShutdownHookManager: Shutdown hook called",
        "18/10/10 06:10:55 INFO ShutdownHookManager: Deleting directory /tmp/spark-6e254b5c-50f9-40d0-af06-c37d8c1a6428",
        "18/10/10 06:10:55 INFO ShutdownHookManager: Deleting directory /tmp/spark-253eac26-e86e-4699-82e6-ad281702fb85",
        "\nstderr: ",
        "\nYARN Diagnostics: "
      ],
      "state": "success"
    }

**Note**: The example above used Kerberized cluster; therefore, `--negotiate` option is used in `curl`.

**Note**: Local directories are disallowed by default for security reasons. Set the local directory to `livy.file.local-dir-whitelist` to allow a specific directory to allow to read. To avoid this setting, please use HDFS path after uploading the JAR and zipped file.

**Note**: To use HWC with Livy in a secure cluster please follow the documentation [here](https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/using-zeppelin/content/using_spark_hwc_and_shc_client_jar_files_with_livy.html).

## 6\. Limitations

* Missing SQL support including USING syntax: currently Data Source V2 implementation [does not support SQL syntax including USING syntax](https://issues.apache.org/jira/browse/SPARK-25280)
* Missing R library support: this library currently supports only Scala, Java, and Python APIs.
* Limitations in Apache Arrow integration and Data Source V2 are inherited. See [SPARK-22386](https://issues.apache.org/jira/browse/SPARK-22386) for Data Source V2 and [SPARK-21187](https://issues.apache.org/jira/browse/SPARK-21187) for Apache Arrow integration in Apache Spark to track ongoing efforts and limitations.
* Supported/unsupported data types and their mapping between Apache Hive and Apache Spark can be found at this [link](https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/integrating-hive/content/hive_hivewarehouseconnector_supported_types.html).

## 7\. Appendix

* [HDP 3.0.1 documentation](https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/integrating-hive/content/hive_hivewarehouseconnector_for_handling_apache_spark_data.html)
* [Hive Warehouse Connector Github](https://github.com/hortonworks/hive-warehouse-connector-release/tree/HDP-3.0.1.0-187-tag)
* Apache Ambari UI for enabling ‘Interactive Query’ in Hive![](https://lh6.googleusercontent.com/MFBoV6fPIIWLfzo53mP7FUOkeeFk8fV5rWIAuRcHx5uhDgxG78-o688V71-CaNwQ5iARopLuwfkORisS-iYCLj1T-TMg_WPZf7XiBOeVwXdqcHir0_ADPjiGjILW9FqTWqgeg7Dv)
* [Introducing Row/ Column Level Access Control for Apache Spark](https://community.hortonworks.com/articles/101181/rowcolumn-level-security-in-sql-for-apache-spark-2.html)
* `SQL INSERT` queries to insert <https://ucr.fbi.gov/crime-in-the-u.s/2016/crime-in-the-u.s.-2016/topic-pages/tables/table-1>

    CREATE DATABASE hwc_db;
    USE hwc_db;
    CREATE TABLE crimes(year INT, crime_rate DOUBLE);
    INSERT INTO crimes VALUES (1997, 611.0), (1998, 567.6), (1999, 523.0), (2000, 506.5), (2001, 504.5), (2002, 494.4), (2003, 475.8);
    INSERT INTO crimes VALUES (2004, 463.2), (2005, 469.0), (2006, 479.3), (2007, 471.8), (2008, 458.6), (2009, 431.9), (2010, 404.5);
    INSERT INTO crimes VALUES (2011, 387.1), (2012, 387.8), (2013, 369.1), (2014, 361.6), (2015, 373.7), (2016, 386.3);