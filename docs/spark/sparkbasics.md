# Apache Spark

## **Mapreduce Vs Spark**

| MapReduce                         | Spark                   |
|----------------------------------|-------------------------|
| Batch Processing                 | Speed                   |
| Complexity                       | Powerful Caching        |
| Data Movement                    | Deployment              |
| Fault Tolerance                  | Real-Time Processing    |
| No Support for Interactive Processing | Polyglot           |
| Not Optimal for Small Files      | Scalability             |

## **Spark Architecture**

![Steps](sparkarc.svg)

> --- **What happens behind the scenes**

1. You launch the application.
2. Spark creates a SparkContext in the Driver.
3. Spark connects to the Cluster Manager (e.g., YARN, standalone, k8s).
4. Cluster Manager allocates Workers and starts Executors.
5. RDD transformations are converted into a DAG (Directed Acyclic Graph).
6. Spark creates Stages, breaks them into Tasks (based on partitions).
7. Tasks are shipped to Executors.
8. Executors run the tasks and return results back to the Driver.
9. Final results (e.g., word count) are written to HDFS.

## **Transformations**

In Spark, a transformation is an operation applied on an RDD (Resilient Distributed Dataset) or DataFrame/Dataset to create a new RDD or DataFrame/Dataset.

Transformations refer to any processing done on data. They are operations that create a new DataFrame (or RDD) from an existing one, but they do not execute immediately. Spark is based on lazy evaluation, meaning transformations are only executed when an action is triggered.

Transformations in Spark are categorized into two types: narrow and wide transformations.

![Steps](trans.svg)

>--- **Narrow Transformations**

In these transformations, all elements that are required to compute the records in a single partition live in the same partition of the parent RDD. Data doesn't need to be shuffled across partitions.

These are transformations that do not require data movement between partitions. In a distributed setup, each executor can process its partition of data independently without needing to communicate with other executors

!!! example
    map, filter, flatmap, sample

>---  **Wide Transformations**

These transformations will have input data from multiple partitions. This typically involves shuffling all the data across multiple partitions.

These transformations require data movement or "shuffling" between partitions. This means an executor might need data from another executor's partition to complete its computation. This data movement makes wide transformations expensive operations

!!! Example
    groupbykey, reducebykey, join, distinct, coalesce, repartition

## **Actions**

Actions in Apache Spark are operations that provide non-RDD values; they return a final value to the driver program or write data to an external system. Actions trigger the execution of the transformation operations accumulated in the Directed Acyclic Graph (DAG).

Actions are operations that trigger the execution of all previous transformations and produce a result. When an action is hit, Spark creates a job.

!!! Example
    collect, count, save, show

> --- **Read & Write operation in Spark are Transformation/Action?**

Reading and writing operations in Spark are often viewed as actions, but they're a bit unique.

Read Operation:Transformations , especially read operations can behave in two ways according to the arguments you provide

!!! Note
    - Lazily evaluated - It will be performed only when an action is called.
    - Eagerly evaluated - A job will be triggered to do some initial evaluations. In case of read.csv()

If it is called without defining the schema and inferSchema is disabled, it determines the columns as string types and it reads only the first line to determine the names (if header=True, otherwise it gives default column names) and the number of fields. 

Basically it performs a collect operation with limit 1, which means one new job is created instantly

Now if you specify inferSchema=True, Here above job will be triggered first as well as one more job will be triggered which will scan through entire record to determine the schema, that's why you are able to see 2 jobs in spark UI

Now If you specify schema explicitly by providing StructType() schema object to 'schema' argument of read.csv(), then you can see no jobs will be triggered here. This is because, we have provided the number of columns and type explicitly and catalogue of spark will store that information and now it doesn't need to scan the file to get that information and this will be validated lazily at the time of calling action.

Writing or saving data in Spark, on the other hand, is considered an action. Functions like saveAsTextFile(), saveAsSequenceFile(), saveAsObjectFile(), or DataFrame write options trigger computation and result in data being written to an external system.

## **DAG**

Spark represents a sequence of transformations on data as a DAG, a concept borrowed from mathematics and computer science. A DAG is a directed graph with no cycles, and it represents a finite set of transformations on data with multiple stages. The nodes of the graph represent the RDDs or DataFrames/Datasets, and the edges represent the transformations or operations applied.

Each action on an RDD (or DataFrame/Dataset) triggers the creation of a new DAG. The DAG is optimized by the Catalyst optimizer (in case of DataFrame/Dataset) and then it is sent to the DAG scheduler, which splits the graph into stages of tasks.

>--- **Job, Stage and Task in Spark**

![Steps](job.svg)

An application in Spark refers to any command or program that you submit to your Spark cluster for execution.
Typically, one spark-submit command creates one Spark application. You can submit multiple applications, but each spark-submit initiates a distinct application.

- Job: 
Within an application, jobs are created based on "actions" in your Spark code.
An action is an operation that triggers the computation of a result, such as collect(), count(), write(), show(), or save().
If your application contains five actions, then five separate jobs will be created.
Every job will have a minimum of one stage and one task associated with it.

- Stage:
A job is further divided into smaller parts called stages.
Stages represent a set of operations that can be executed together without shuffling data across the network. Think of them as logical steps in a job's execution plan. Stages are primarily defined by "wide dependency transformations".

- Task:
A task is the actual unit of work that is executed on an executor.
It performs the computations defined within a stage on a specific partition of data.
The number of tasks within a stage is directly determined by the number of partitions the data has at that point in the execution. If a stage operates on 200 partitions, it will typically launch 200 tasks.

## **Lazy Evaluation**

Lazy evaluation in Spark means that the execution doesn't start until an action is triggered. In Spark, transformations are lazily evaluated, meaning that the system records how to compute the new RDD (or DataFrame/Dataset) from the existing one without performing any transformation. The transformations are only actually computed when an action is called and the data is required. 

!!! example

    spark.read.csv() 
    will not actually read the data until an action like .show() or .count() is performed

## **SparkSession vs SparkContext**

Both Spark Session and Spark Context serve as the entry point into a Spark cluster, similar to how a `main` method serves as the entry point for code execution in languages like C++ or Java. This means that to run any Spark code, you first need to establish one of these entry points.

> --- **Spark Session**

The Spark Session is the unified entry point introduced in Spark 2.0. It is now the primary way to interact with Spark.

Prior to Spark 2.0 (specifically up to Spark 1.4), if you wanted to work with different Spark functionalities like SQL, Hive, or Streaming, you had to create separate contexts for each (e.g., `SQLContext`, `HiveContext`, `StreamingContext`). The Spark Session encapsulates all these different contexts, providing a single object to access them. This simplifies development as you only need to create a Spark Session to gain access to all necessary functionalities.

If you've been using Databricks notebooks, you might have implicitly been using a Spark Session without realizing it. Databricks typically provides a default `spark` object, which is an instance of `SparkSession`, allowing you to directly write code like `spark.read.format(...)`. This is why the local setup is demonstrated in the source, as the default session is not automatically provided outside environments like Databricks.

You can configure properties like `spark.driver.memory` by using the `.config()` method when building the Spark Session.

> --- **Spark Context**

The Spark Context (`SparkContext`) was the original entry point for Spark applications before Spark 2.0.

In earlier versions of Spark (up to Spark 1.4), `SparkContext` was the primary entry point for general Spark operations. However, for specific functionalities like SQL, you needed additional context objects like `SQLContext`.

While Spark Session has become the dominant entry point, `SparkContext` is still relevant for RDD (Resilient Distributed Dataset) level operations. If you need to perform low-level transformations directly on RDDs (e.g., `flatMap`, `map`), you would typically use the Spark Context. An example provided is writing a word count program using RDDs, where `SparkContext` comes into use.

With the advent of Spark Session, you do not create a `SparkContext` directly as a separate entry point anymore. Instead, you can access the `SparkContext` object through the `SparkSession` instance. This means that the `SparkContext` is now encapsulated within the `SparkSession`.

> --- **Code Example**

Here’s an example demonstrating how to create a Spark Session and then obtain a Spark Context from it, based on the provided transcript:

```python
from pyspark.sql import SparkSession
spark_builder = SparkSession.builder
spark_session_config = spark_builder.master("local").appName("Testing").config("spark.driver.memory", "12g")
spark = spark_session_config.getOrCreate()
print(spark)
sc = spark.sparkContext
print(sc)
```

## Readmodes

```python
flight_df = spark.read.format("csv") \
                .option("header", "false") \
                .option("inferschema","false")\
                .option("mode","FAILFAST")\
                .load("flightdata.csv") 
```

If a CSV file contains a header but you are defining a manual schema, you might need to skip the first row to prevent the header text (which is a string) from causing errors in numeric columns (like "count" being an integer).

!!! Note
    `skipRows` option: This allows you to bypass a specific number of rows at the top of the file.

    If you use `mode("failFast")` and your manual schema expects an Integer but finds a String header, the job will crash. 
    
    Using `mode("permissive")` will instead turn those mismatched values into nulls, allowing you to see the data while you debug.

| Mode           | Description                                                                 |
|----------------|-----------------------------------------------------------------------------|
| failFast   | Terminates the query immediately if any malformed record is encountered. This is useful when data integrity is critical. |
| dropMalformed | Drops all rows containing malformed records. This can be useful when you prefer to skip bad data instead of failing the entire job. |
| permissive (default) | Tries to parse all records. If a record is corrupted or missing fields, Spark sets `null` values for corrupted fields and puts malformed data into a special column named `_corrupt_record`.|


## **RDD**

RDDs are the building blocks of any Spark application.

RDDs Stands for

- Resilient: Fault tolerant and is capable of rebuilding data on failure
- Distributed: Distributed data among the multiple nodes in a cluster
- Dataset: Collection of partitioned data with values

RDD is the fundamental data structure of Spark, which allows it to efficiently operate on large-scale data across a distributed environment.

Once an RDD is created, it cannot be changed. Any transformation applied to an RDD creates a new RDD, leaving the original one untouched.

RDDs are fault-tolerant, meaning they can recover from node failures. This resilience is provided through a feature known as lineage, a record of all the transformations applied to the base data.

RDDs follow a lazy evaluation approach, meaning transformations on RDDs are not executed immediately, but computed only when an action (like count, collect) is performed. This leads to optimized computation.

> --- **When to Use RDDs (Advantages)**

Despite the general recommendation to use DataFrames/Datasets, RDDs have specific use cases where they are advantageous:

1. Unstructured Data
2. Full Control and Flexibility
3. Type Safety (Compile-Time Errors)

> --- **Why You Should NOT Use RDDs (Disadvantages)**

1. No Automatic Optimization by Spark
2. Complex and Less Readable Code
3. Potential for Inefficient Operations
4. Developer Burden

> --- **Difference**

| Criteria         | RDD (Resilient Distributed Dataset)                                           | DataFrame                                                                                    | DataSet                                                                                    |
| -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Abstraction      | Low level, provides a basic and simple abstraction.                               | High level, built on top of RDDs. Provides a structured and tabular view on data.                | High level, built on top of DataFrames. Provides a structured and strongly-typed view on data. |
| Type Safety      | Provides compile-time type safety, since it is based on objects.                  | Doesn't provide compile-time type safety, as it deals with semi-structured data.                 | Provides compile-time type safety, as it deals with structured data.                           |
| Optimization     | Optimization needs to be manually done by the developer (like using `mapreduce`). | Makes use of Catalyst Optimizer for optimization of query plans, leading to efficient execution. | Makes use of Catalyst Optimizer for optimization.                                              |
| Processing Speed | Slower, as operations are not optimized.                                          | Faster than RDDs due to optimization by Catalyst Optimizer.                                      | Similar to DataFrame, it's faster due to Catalyst Optimizer.                                   |
| Ease of Use      | Less easy to use due to the need of manual optimization.                          | Easier to use than RDDs due to high-level abstraction and SQL-like syntax.                       | Similar to DataFrame, it provides SQL-like syntax which makes it easier to use.                |
| Interoperability | Easy to convert to and from other types like DataFrame and DataSet.               | Easy to convert to and from other types like RDD and DataSet.                                    | Easy to convert to and from other types like DataFrame and RDD.                                |

## **Spark Query Plan**

The Spark SQL Engine is fundamentally the Catalyst Optimizer. Its primary role is to convert user code (written in DataFrames, SQL, or Datasets) into Java bytecode for execution. This conversion and optimization process occurs in four distinct phases. It's considered a compiler because it transforms your code into Java bytecode. It plays a key role in optimizing code leveraging concepts like lazy evaluation

![Steps](sparksqlengine.svg)

### **Phase 1: Unresolved Logical Plan**

This is the initial stage where you write your code using DataFrames, SQL, or Datasets APIs.

When you write transformations (e.g., select, filter, join), Spark creates an "unresolved logical plan". This plan is like a blueprint or a "log" of transformations, indicating what operations need to be performed in what order.

At this stage, the plan is "unresolved" because Spark has not yet checked if the tables, columns, or files referenced actually exist

### **Phase 2: Analysis**

To resolve the logical plan by checking the existence and validity of all referenced entities.

This phase heavily relies on the Catalog. The Catalog is where Spark stores metadata (data about data). It contains information about tables, files, databases, their names, creation times, sizes, column names, and data types. For example, if you read a CSV file, the Catalog knows its path, name, and column headers.

The Analysis phase queries the Catalog to verify if the files, columns, or tables specified in the unresolved logical plan actually exist.

If everything is found and validated, the plan becomes a "Resolved Logical Plan". If any entity is not found (e.g., a non-existent file path or a misspelled column name), Spark throws an AnalysisException

### **Phase 3: Logical Optimization**

To optimize the "Resolved Logical Plan" without considering the physical execution aspects. It focuses on making the logical operations more efficient.

!!! Note

    Predicate Pushdown: If you apply multiple filters, the optimizer might combine them or push them down closer to the data source to reduce the amount of data processed early.
 
    Column Pruning: If you select all columns (SELECT *) but then only use a few specific columns in subsequent operations, the optimizer will realize this and modify the plan to only fetch the necessary columns from the start, saving network I/O and processing.

This phase benefits from Spark's lazy evaluation, allowing it to perform these optimizations before any actual computation begins. An "Optimized Logical Plan"

### **Phase 4: Physical Planning**

The "Optimized Logical Plan" is converted into multiple possible "Physical Plans". Each physical plan represents a different strategy for executing the logical operations (e.g., different join algorithms).

Spark applies a "Cost-Based Model" to evaluate these physical plans. It estimates the resources (memory, CPU, network I/O) each plan would consume if executed.

The plan that offers the best resource utilization and lowest estimated cost (e.g., least data shuffling, fastest execution time) is selected as the "Best Physical Plan".

!!! Example
  
    For joins, if one table is significantly smaller than the other, Spark might choose a Broadcast Join. This involves sending the smaller table to all executor nodes where the larger table's partitions reside. This avoids data shuffling (expensive network operations) of the larger table across the cluster, leading to significant performance gains.

The Best Physical Plan, which is essentially a set of RDDs (Resilient Distributed Datasets) ready to be executed on the cluster.

### **Phase 5:Whole-Stage Code Generation**

This is the final step where the "Best Physical Plan" (the RDD operations) is translated into Java bytecode.

This bytecode is then sent to the individual executors on the cluster to be executed. This direct bytecode generation improves performance by eliminating interpretation overhead and allowing the JVM to further optimize the code.

> --- **In what cases will predicate pushdown not work?**

- Complex Data Types:

Spark's Parquet data source does not push down filters that involve complex types, such as arrays, maps, and struct. This is because these complex data types can have complicated nested structures that the Parquet reader cannot easily filter on.

Here's an example:

```
root
 |-- Name: string (nullable = true)
 |-- properties: map (nullable = true)
 |    |-- key: string
 |    |-- value: string (valueContainsNull = true)

+----------+-----------------------------+
|Name      |properties                   |
+----------+-----------------------------+
|Afaque    |[eye -> black, hair -> black]|
|Naved     |[eye ->, hair -> brown]      |
|Ali       |[eye -> black, hair -> red]  |
|Amaan     |[eye -> grey, hair -> grey]  |
|Omaira    |[eye -> , hair -> brown]     |
+----------+-----------------------------+
```

```python
df.filter(df.properties.getItem("eye") == "brown").show()
```

```
== Physical Plan ==
*(1) Filter (metadata#123[key] = value)
+- *(1) ColumnarToRow
   +- FileScan parquet [id#122,metadata#123] Batched: true, DataFilters: [(metadata#123[key] = value)], Format: Parquet, ...
```

- Unsupported Expressions: 

In Spark, `Parquet` data source does not support pushdown for filters involving a `.cast` operation.

The reason for this behaviour is as follows: `.cast` changes the datatype of the column, and the Parquet data source may not be able to perform the filter operation correctly on the cast data.

!!! Note

    This behavior may vary based on the data source. For example, if you're working with a JDBC data source connected to a database that supports SQL-like operations, the `.cast` filter could potentially be pushed down to the database.


## **Data Type & Schema**

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

# Defining the manual schema
my_schema = StructType([
    StructField("destination_country", StringType(), True),
    StructField("origin_country", StringType(), True),
    StructField("count", IntegerType(), True)
])

# Using the schema to read a CSV file
flight_df = spark.read \
    .format("csv") \
    .schema(my_schema) \
    .option("header", "false") \
    .option("mode", "permissive") \
    .load("/FileStore/tables/flight_data.csv")

flight_df.show(5)
```

There are two primary methods: using StructType and StructField classes, and using a DDL (Data Definition Language) string.

These are classes in Spark used to define the schema structure.

StructField represents a single column within a DataFrame. It holds information such as the column's name, its data type (e.g., String, Integer, Timestamp), and whether it can contain null values (nullable: True/False). If nullable is set to False, the column cannot contain NULL values, and an error will be thrown if it does.

StructType defines the overall structure of a DataFrame. It is essentially a list or collection of StructField objects.

!!! Tip
    What happens if you set header=False when your data actually has a header? If you disable the header option (header=False) but your CSV file contains a header row, Spark will treat that header row as regular data.
    If this header row's values do not match the data types defined in your manual schema (e.g., a string "Count" being read into an Integer column), it can lead to null values in that column if the read mode is set to permissive, or an error if the mode is failfast

## **Json read**

```python
# Reading Multi-line JSON
df = spark.read \
    .format("json") \
    .option("multiLine", "true") \
    .load("/path/to/multiline_file.json")

df.show()
```

Line-Delimited (Default): Each line in the file represents exactly one JSON record. This is faster because Spark reads and processes the file line-by-line.

Multi-Line: A single JSON record spans multiple lines (often formatted with indents for readability). This is slower because Spark must treat the entire file as a single object and parse it to find where records start and end.

##  **Handling corrupted records**

When reading data, Spark offers different modes to handle corrupted records, which influence how the DataFrame is populated.

In permissive mode, all records are allowed to enter the DataFrame. If a record is corrupted, Spark sets the malformed values to null and does not throw an error. For the example data with five total records (two corrupted), permissive mode will result in five records in the DataFrame, with nulls where data is bad.

In dropMalformed mode, Spark discards any record it identifies as corrupted.

In failfast mode, Spark immediately throws an error and stops the job as soon as it encounters the first corrupted record. This mode will result in zero records in the DataFrame because the job will fail.

## **Print bad records**

In production, where thousands of records might be corrupted, printing them is inefficient

Spark provides an option to store these records in a specified directory for later audit

```
df_stored = spark.read \
    .format("csv") \
    .option("header", "true") \
    .option("badRecordsPath", "/FileStore/tables/bad_records_output") \
    .schema(emp_schema) \
    .load("/path/to/employee.csv")
```

To specifically identify and view the corrupted records, you need to define a manual schema that includes a special column named _corrupt_record. This column will capture the raw content of the corrupted record.

Where to store bad record For scenarios with a large volume of corrupted records (e.g., thousands), printing them is not practical. Spark provides the badRecordsPath option to store all corrupted records in a specified location. These records are saved in JSON format at the designated path.

## **Write Modes**

```python
df.write \
  .format("parquet") \
  .mode("overwrite") \
  .partitionBy("date") \
  .save("/data/output/path")
```

The mode() method in the DataFrame Writer API is crucial as it dictates how Spark handles existing data at the target location. There are four primary modes:

- append: If files already exist at the specified location, the new data from the DataFrame will be added to the existing files.

- overwrite: This mode deletes any existing files at the target location before writing the new DataFrame.

- errorIfExists: Spark will check if a file or location already exists at the target path. If it does, the write operation will fail and throw an error. Useful when you want to ensure that you do not accidentally overwrite or append to existing data.

- ignore: If a file or location already exists at the target path, Spark will skip the write operation entirely without throwing an error. The new file will not be written. This mode is suitable if you want to prevent new data from being written if data is already present, perhaps to avoid overwriting changes or to ensure data integrity


## **Spark Submit**

Spark Submit is a command-line tool that allows you to trigger or run your Spark applications on a Spark cluster. It packages all the required files and JARs (Java Archive files) and deploys them to the Spark cluster for execution. It is used to run jobs on various types of Spark clusters

> --- **Where is Your Spark Cluster Located?**

Spark clusters can be deployed in multiple environments. When using Spark Submit, you specify the location of your master node.
   
!!! Example
    
    spark-submit  
    --master {stanadlone,yarn.mesos,kubernetes}  
    --deploy-mode {client/cluster}  
    --class mainclass.scala   
    --jars mysql-connector.jar   
    --conf spark.dynamicAllocation.enabled=true    
    --conf spark.dynamicAllocation.minExecutors=1    
    --conf spark.dynamicAllocation.maxExecutors=10    
    --conf spark.sql.broadcastTimeout=3600    
    --conf spark.sql.autobroadcastJoinThreshold=100000    
    --conf spark.executor.cores=2    
    --conf spark.executor.instances=5    
    --conf spark.default.parallelism=20    
    --conf spark.driver.maxResultSize=1G    
    --conf spark.network.timeout=800   
    --conf spark.driver.maxResultSize=1G    
    --conf spark.network.timeout=800    
    --driver-memory 1G    
    --executor-memory 2G    
    --num-executors 5    
    --executor-cores 2    
    --py-files /path/to/other/python/files.zip 
    /path/to/your/python/wordcount.py    /path/to/input/textfile.txt 

Key arguments include --master, --deploy-mode, resource configs like --executor-memory, --num-executors, dependency options like --jars and --packages, and --conf for tuning Spark properties. These control execution environment, resource allocation, and dependencies.”

## **Deployment modes**

![Steps](sparkmode.svg)

> --- **Client Mode**

![Steps](sparkclient.svg)

In Client mode, the Spark Driver runs directly on the edge node (or the machine from which the spark-submit command is executed).

The Executors, however, still run on the worker nodes within the cluster.

- Advantages:
    - Easy Debugging and Real-time Logs: Logs (STD OUT and STD ERR) are generated directly on the client machine (edge node). This makes it very easy for developers to monitor the process, see real-time output, debug issues, and observe errors as they occur. This mode is highly suitable for development and testing of small code snippets.
- Disadvantages:
    - Vulnerability to Edge Node Shutdown: If the edge node is shut down, either accidentally or intentionally, the Spark Driver (running on it) will be terminated. Since the Driver coordinates the entire application, its termination will cause all associated Executors to be killed, leading to the entire Spark job stopping abruptly and incompletely.
    - High Network Latency: Communication between the Driver (on the edge node) and the Executors (on worker nodes in the cluster) involves two-way communication across the network. This can introduce network latency, especially for operations like Broadcaster, where data needs to be first sent to the Driver and then distributed to Executors.
    - Potential for Driver Out of Memory (OOM) Errors: If multiple users submit jobs in Client mode from the same edge node, and their collective Driver memory requirements exceed the edge node's physical memory capacity (which is typically lower than worker nodes), processes may fail to start or encounter Driver OOM errors

When u start a spark shell, application driver creates the spark session in your local machine which request to Resource Manager present in cluster to create Yarn application. YARN Resource Manager start an Application Master (AM container). For client mode Application Master acts as the Executor launcher. Application Master will reach to Resource Manager and request for further containers. 
Resource manager will allocate new containers. These executors will directly communicate with Drivers which is present in the system in which you have submitted the spark application.

> --- **Cluster Mode**

![Steps](sparkcluster.svg)

For cluster mode, there’s a small difference compare to client mode in place of driver. Here Application Master will create driver in it and driver will reach to Resource Manager.

In Cluster mode, the Spark Driver (Application Master container) is launched and runs on one of the worker nodes within the Spark cluster. The Executors also run on other worker nodes in the cluster.

- Advantages:
    - Resilience and Disconnect-ability: Once a Spark job is submitted in Cluster mode, the Driver runs independently within the cluster. This means the user can disconnect from or even shut down their edge node machine without affecting the running Spark application. This makes it ideal for long-running jobs.
    - Low Network Latency: Both the Driver and the Executors are running within the same cluster. This proximity significantly reduces network latency between them, leading to more efficient data transfer and communication.
    - Scalability and Resource Utilization: Worker nodes are provisioned with significant memory and processing capabilities. By running the Driver on a worker node, the application can leverage the cluster's robust resources, reducing the likelihood of Driver OOM issues, even with many concurrent jobs.
    - Suitable for Production Workloads: Cluster mode is the recommended deployment mode for production workloads, especially for scheduled jobs that run automatically and do not require constant real-time monitoring on the client side.
- Disadvantages:
    Indirect Log Access: Logs and output are not directly displayed on the client machine. When a job is submitted in Cluster mode, an Application ID is generated. Users must use this Application ID to access the Spark Web UI (User Interface) to track the job's status, progress, and logs. This adds an extra step for monitoring compared to Client mode

> --- **Local Mode**

![Steps](localmode.svg)

In local mode, Spark runs on a single machine, using all the cores of the machine. It is the simplest mode of deployment and is mostly used for testing and debugging.

> --- **Comparison**

| Feature              | Client Mode                                      | Cluster Mode                                    |
| -------------------- | ------------------------------------------------ | ----------------------------------------------- |
| Driver Location      | Edge Node (or client machine)                    | Worker Node within the cluster                  |
| Log Generation       | On client machine (STD OUT, STD ERR)             | Application ID generated; view via Spark Web UI |
| Debugging            | Easy, real-time feedback                         | Requires checking Spark Web UI                  |
| Network Latency      | High (Driver <-> Executors across network)       | Low (Driver <-> Executors within cluster)       |
| Edge Node Shutdown   | Application stops (Driver killed)                | Application continues to run                    |
| Driver Out of Memory | Higher chance if many users/low edge node memory | Lower chance (cluster has more resources)       |
| Use Case             | Development, small code snippets, debugging      | Production workloads, long-running jobs         |
