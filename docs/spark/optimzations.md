## **Partitioning**

Partitioning and Bucketing are data optimization techniques used in Spark during the data writing process to improve performance for future read operations, joins, and filters. By deciding how data is organized on disk, you can significantly reduce the amount of data Spark needs to scan.

> --- **Partitioning**

Partitioning organizes data into a hierarchical folder structure based on the distinct values of one or more columns. 

For every unique value in the partitioned column, Spark creates a separate folder. For example, partitioning by "Address" (Country) would create folders like `Address=India`, `Address=USA`, etc..

When you query data using a filter (e.g., `WHERE Country = 'India'`), Spark skip all other folders and only reads the relevant one, avoiding a full table scan.

You can partition by multiple columns. The order of columns matters; for example, partitioning by `Address` then `Gender` creates gender folders inside each country folder.


```python
# Partitioning by a single column (Address)
df.write \
  .partitionBy("Address") \
  .format("csv") \
  .save("/path/to/destination")

# Partitioning by multiple columns (Address and then Gender)
df.write \
  .partitionBy("Address", "Gender") \
  .format("csv") \
  .save("/path/to/destination_multi")
```

Partitioning is inefficient for high-cardinality columns (columns with many unique values, like a User ID). If you partition by ID, Spark might create millions of tiny files, which degrades performance.

## **Bucketing**

Bucketing is used when partitioning is not suitable, particularly for high-cardinality columns or to optimize joins.

Spark uses a hash function on a specific column to distribute data into a fixed number of "buckets" (files).

Unlike partitioning, bucketing is not supported by the standard `.save()` method on file systems; it requires using `saveAsTable` because the metadata must be stored in the Hive Metastore.

If two tables are bucketed on the same column with the same number of buckets, Spark can perform a join without "shuffling" data across the network, which is a very expensive operation.

Spark knows exactly which bucket contains a specific value (e.g., a specific Aadhaar ID), allowing it to search only a small fraction of the data.

```python
# Bucketing by ID into 3 buckets
df.write \
  .bucketBy(3, "ID") \
  .saveAsTable("bucketed_table")
```

If you have 200 tasks running and you ask for 5 buckets, Spark might create $200 \times 5 = 1,000$ files. To prevent this, repartition the data to match the number of buckets before writing.

```python
# Optimize by matching partitions to buckets
df.repartition(5) \
  .write \
  .bucketBy(5, "ID") \
  .saveAsTable("optimized_bucket_table")
```

> --- **Summary Comparison**

| Feature | Partitioning | Bucketing |
| :--- | :--- | :--- |
| Logic | Groups data into folders based on column values. | Groups data into a fixed number of files via hashing. |
| Best For | Low-cardinality columns (e.g., Country, Gender). | High-cardinality columns (e.g., ID) and Join optimization. |
| Output | Created as directory structures on the file system. | Created as specific bucketed files within a table. |
| Constraint | Can lead to "too many small files" if cardinality is high. | Must be saved as a table (`saveAsTable`). |

The need for repartition and coalesce arises from issues faced when processing large datasets in Spark, particularly concerning data partitioning.

![Steps](repart.svg)

When a DataFrame is created, it's often divided into multiple partitions. Sometimes, these partitions can be of uneven sizes (e.g., 10MB, 20MB, 40MB, 100MB).

Processing smaller partitions (e.g., 10MB) takes less time than larger ones (e.g., 100MB). This leads to idle Spark executors: while one executor is busy with a large partition, others might finish their tasks quickly and then wait for the large partition to complete. This causes time delays and underutilization of allocated resources (e.g., RAM)

This situation often arises after operations like join transformations. For instance, if a join operation is performed on a product column, and one product is a "best-selling product" with a high number of records, all those records might get grouped into a single partition, making it very large. This phenomenon is called data skew.

Users often see messages like "199 out of 200 partitions processed," where the last remaining partition takes a significantly longer time to complete due to its large size.

To deal with these scenarios and optimize performance, Spark provides repartition and coalesce methods

## **Repartition**

Repartition shuffles the entire dataset across the cluster. This means data from existing partitions can be moved to new partitions.

The primary goal of repartition is to evenly distribute data across the specified number of new partitions. For example, if you have 200MB of data across five uneven partitions and repartition it into five, it will aim for 40MB per partition.

Repartition can increase or decrease the number of partitions. If you initially have 5 partitions but need 10, repartition is the only choice.It can be used when you want to increase the number of partitions to allow for more concurrent tasks and increase parallelism when the cluster has more resources.

Due to the shuffling operation, repartition is generally more expensive and involves more I/O operations compared to coalesce.

Pros and Cons of Repartition are Evenly distributed data. More I/O (Input/Output) because of shuffling. More expensive.

In certain scenarios, you may want to partition based on a specific key to optimize your job. For example, if you frequently filter by a certain key, you might want all records with the same key to be on the same partition to minimize data shuffling. In such cases, you can use repartition() with a column name.

## **Coalesce**

Coalesce merges existing partitions to reduce the total number of partitions.

Crucially, coalesce tries to avoid full data shuffling. It achieves this by moving data from some partitions to existing ones, effectively merging them locally on the same executor if possible. This makes it less expensive than repartition.

Because it avoids full shuffling, coalesce does not guarantee an even distribution of data across the new partitions. It might result in an uneven distribution, especially if the original partitions were already skewed.

Coalesce can only decrease the number of partitions. It cannot be used to increase the number of partitions. If you need more partitions, you must use repartition.

Pros and Cons of Coalesce: No shuffling (or minimal shuffling). Not expensive (cost-effective). Uneven data distribution.

However, it can lead to  data skew if you have fewer partitions than before, because it combines existing partitions to reduce the total number.

>--- **When to Choose Which?**

The choice between repartition and coalesce is use-case dependent

**Choose repartition when**:

 You need to evenly distribute data across partitions, which is crucial for balanced workload across executors.

 You need to increase the number of partitions (e.g., if you have too few partitions or want to process data in smaller chunks in parallel).

 You are okay with the overhead of a full shuffle, as the benefit of even distribution outweighs the cost.

 Dealing with severe data skew is a primary concern.

**Choose coalesce when**:

 You primarily need to decrease the number of partitions (e.g., after filter operations drastically reduce data, or before writing to a single file).

 You want to minimize shuffling and I/O costs.

 You can tolerate slightly uneven data distribution across partitions, or the data skew is minimal and won't significantly impact performance.

 You want to save processing time and cost by avoiding a full shuffle

> --- **Why doesn't `.coalesce()` explicitly show the partitioning scheme?**

`.coalesce` doesn't show the partitioning scheme e.g. `RoundRobinPartitioning` because the operation only minimizes data movement by merging into fewer partitions, it doesn't do any shuffling. Because no shuffling is done, the partitioning scheme remains the same as the original DataFrame and Spark doesn't include it explicitly in it's plan as the partitioning scheme is unaffected by `.coalesce`

## **Spark Strategy Joins**

It is important to distinguish between Join Types and Join Strategies.

Join Types refers to the logical result of the join (e.g., Left Join, Right Join, Inner Join, etc.).

Join Strategies refers to the internal implementation or "strategy" Spark uses to execute the join across a cluster (e.g., Shuffle Sort Merge Join, Broadcast Join).

> --- **Why Joins are Expensive: The Shuffling Process**

Joins are considered "expensive" operations in Spark because they often require shuffling (or "saapling" as referred to in the source), which involves moving data across the network between executors.

If you have two DataFrames of 500MB each with a default HDFS block size of 128MB, Spark will create 4 partitions for each DataFrame. These partitions are distributed across different executors/worker nodes.

To join data on a specific key (e.g., `ID`), the data for that same key must reside on the same executor. If `ID 1` is on Executor A and its corresponding match is on Executor B, Spark must move that data to a common location.

For wide transformations like joins, Spark defaults to creating 200 partitions.

Spark uses a hash-based approach to determine where data goes. For example, it might calculate `ID % 200` to determine the partition number. This ensures that the same ID from both DataFrames always ends up in the same partition.

Conceptual Code Example (Shuffling):
```python
# Spark defaults to 200 partitions for shuffle operations
spark.conf.set("spark.sql.shuffle.partitions", "200")

# Performing a join triggers shuffling to align keys across the cluster
df_joined = df1.join(df2, df1.id == df2.id, "inner")
```

Spark utilizes five primary strategies to perform joins:

1. Shuffle Sort Merge Join (SSMJ)
2. Shuffle Hash Join (SHJ)
3. Broadcast Hash Join (BHJ)
4. Cartesian Join
5. Broadcast Nested Loop Join (BNLJ)

### **Shuffle Sort Merge Join**

This is one of the default join strategies in Spark for large datasets.

Data with the same join keys are shuffled across the cluster so that matching keys end up in the same partition. Within each partition, the data is sorted based on the join key. Spark then performs a merge operation by iterating over the sorted datasets to find matching records.

Sort Merge Join is a shuffle-based join where data is partitioned and sorted on join keys, then merged. It’s efficient for large datasets and doesn’t require data to fit in memory. It is CPU intensive.

### **Shuffle Hash Join**

In Shuffle Hash Join, data is shuffled so that rows with the same join keys are brought to the same partitions.

Within each partition, Spark builds an in-memory hash table for the smaller side of the join. It then scans the larger dataset and probes the hash table to find matching records.

The lookup phase is O(1), making it faster than Sort Merge Join when data fits in memory. However, it is memory-intensive because the hash table must fit in executor memory. If it exceeds available memory, it may lead to spilling or Out of Memory (OOM) errors.

Shuffle Hash Join builds a hash table on the smaller side after shuffling data, enabling O(1) lookups. It’s faster than Sort Merge Join but requires enough memory.

### **Broadcast Nested Loop Join**

Broadcast Nested Loop Join is one of the most expensive join strategies in Spark. It is typically used when there is no equality condition in the join (non-equi joins), such as `df1.id > df2.id`.

In this strategy, Spark broadcasts the smaller dataset to all executors. Each partition of the larger dataset then compares its rows with every row of the broadcasted dataset.

This behaves like nested loops, resulting in a time complexity close to O(n × m), which can become very expensive for large datasets.

Conceptual Code Example (Non-Equi Join):

```python
df_non_equi = df1.join(df2, df1.id > df2.id, "inner")
```

Broadcast Nested Loop Join is used for non-equi joins. Spark broadcasts the smaller dataset and compares every row of the large dataset with every row of the smaller dataset, which makes it O(n²) and very expensive.

### **Broadcast Join**

Broadcast Hash Join is a specialized join strategy used to optimize performance by eliminating the need for data shuffling,. While standard joins like Shuffle Sort Merge Join (the Spark default) move data across the network to align keys, a Broadcast Join sends the entire smaller dataset to every worker node,.

The core mechanism involves the Driver and the Executors - The Spark Driver identifies a table that is small enough to be broadcast. It must have sufficient memory to store this table locally before distributing it. The Driver sends a complete copy of the small table to every Executor in the cluster.

Once the small table is residing on every Executor, each Executor can perform the join locally using its own partition of the large table. This makes the Executors "self-sufficient" because they no longer need to fetch data from other nodes via shuffling.

> --- **When to Use Broadcast Joins**

It is ideal when you have one large table (e.g., 1GB) and one small table (e.g., less than 10MB). Use it to prevent "cluster choking" caused by moving massive amounts of data across the network. While not the primary focus of this source, the transcript notes that avoiding shuffling is the main goal of this strategy.

> --- **Checking and Setting the Broadcast Threshold**

Spark uses a default threshold to decide if a table should be automatically broadcast. This is typically 10 MB.

```python
# To get the current broadcast threshold (returns value in bytes)
current_threshold = spark.conf.get("spark.sql.autoBroadcastJoinThreshold")
print(current_threshold) # Default is 10485760 (10 MB)

# To change the threshold (e.g., to 20 MB)
# You must convert MB to bytes
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "20971520")

# To disable automatic broadcasting
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
```

> --- **Forcing a Broadcast Join with Hints**

If Spark does not automatically choose a broadcast join, you can provide a hint in your code to force it.

```python
from pyspark.sql.functions import broadcast

# Standard join might result in a Sort Merge Join
# df_joined = df_large.join(df_small, "id")

# Using the broadcast hint to force a Broadcast Hash Join
df_joined = df_large.join(broadcast(df_small), df_large.id == df_small.id, "inner")

# Inspecting the physical plan to verify the join strategy
df_joined.explain() 
```

> --- **Potential Risks and Failure Points**

Despite its efficiency, Broadcast Join can fail in the following scenarios:

Since the small table must first be collected by the Driver, if the table is too large for the Driver's memory, the application will crash. If the Executor's memory is already near capacity, adding a broadcasted table (even a 100MB one) can trigger an OOM during the join operation. Attempting to broadcast a very large file (e.g., 1GB) will saturate the network because that file must be sent to every single executor.

### **Cartesian Join (Cross Join)**

Cartesian Join (also known as Cross Join) produces the Cartesian product of two datasets, meaning every row from the first dataset is combined with every row from the second dataset.

If one dataset has n rows and the other has m rows, the result will contain n × m rows, making it extremely expensive for large datasets.

Spark does not perform Cartesian joins by default unless explicitly enabled or requested, as it can lead to massive data explosion.


```python
df1.crossJoin(df2)
```

>--- **When It Happens**

- When no join condition is specified
- When explicitly using crossJoin()
- When join condition is missing or incorrect

## **Cache**

Caching is an optimization technique in Spark that allows you to store intermediate results in memory. This prevents Spark from re-calculating the same data repeatedly when it is used multiple times in subsequent transformations.

When data is cached, it is stored within the Storage Memory Pool of a Spark Executor. An executor's memory is divided into three parts: User Memory, Spark Memory, and Reserved Memory. Spark Memory, in turn, contains two pools: Storage Memory Pool and Execution Memory Pool. Cached data specifically resides in the Storage Memory Pool. If the Storage Memory Pool fills up, Spark might evict (remove) data that is not frequently used (using an LRU - Least Recently Used - fashion) or spill it to disk.

If a DataFrame (DF) is used multiple times, Spark will re-calculate it from the beginning each time it's referenced, because DataFrames are immutable and executors' memories are short-lived

By calling .cache() on df, its intermediate result is stored in the Storage Memory Pool. Now, whenever df is needed again, Spark directly retrieves it from memory instead of re-calculating it. This significantly reduces computation time and improves efficiency

You should avoid caching data when the DataFrame is very small or when its re-calculation time is negligible. Caching consumes memory, and if the benefits of caching don't outweigh the memory consumption, it's better to avoid it.

>--- **Limitations of Caching**

If the cached data's partitions are larger than the available Storage Memory Pool, the excess partitions will not be stored in memory and will either be re-calculated on the fly or spilled to disk if using a storage level that supports it. Spark does not store partial partitions; a partition is stored entirely or not at all.

If a cached partition is lost (e.g., due to an executor crash), Spark will re-calculate it using the DAG lineage

How to Uncache Data - To remove data from the cache, you can use the .unpersist() method.

When you call df.cache(), it internally calls df.persist() with a default storage level of MEMORY_AND_DISK.

persist() offers more flexibility because it allows you to specify the desired storage level as an argument

## **Persist**

Storage levels define where data is stored (memory, disk, or both) and how it is stored (serialized or deserialized, and with replication). These levels provide fine-grained control over how cached data is managed, balancing performance, fault tolerance, and memory usage.

To use StorageLevel with persist(), you need to import it: from pyspark import StorageLevel

Here are the different storage levels explained:

>--- **MEMORY_ONLY**

Stores data only in RAM (deserialized form). If memory is insufficient, partitions will be re-calculated when needed. Fastest processing because data is in memory and readily accessible. High memory utilization, potentially limiting other operations.

For small to medium-sized datasets that fit entirely in memory and where re-calculation overhead is high.

>--- **MEMORY_AND_DISK**

Default for cache(). Attempts to store data in RAM first (deserialized form).

If RAM is full, excess partitions are spilled to disk (serialized form). Provides a good balance of speed and resilience; data is less likely to be re-calculated.

Disk access is slower than memory. Data read from disk (serialized) requires CPU to deserialize it, leading to higher CPU utilization.

For larger datasets that might not fully fit in memory but where performance is still critical.

>--- **MEMORY_ONLY_SER**

Stores data in RAM only, but in a serialized form.

Serialization saves memory space, allowing more data to be stored in the same amount of RAM (e.g., 5GB uncompressed might become 8GB serialized).

Data needs to be deserialized by the CPU when accessed, leading to higher CPU utilization and slightly slower access compared to MEMORY_ONLY.

This serialization specifically works for Java and Scala objects, and not for Python objects (though Python has its own pickling mechanisms, the _SER storage levels in Spark are typically for JVM objects).

When memory is a major constraint and you can tolerate increased CPU usage for deserialization.

>--- **MEMORY_AND_DISK_SER**

Stores data first in RAM (serialized), then spills to disk (serialized) if memory is full.

Combines memory saving of serialization with resilience of disk storage.

High CPU usage due to deserialization for both memory and disk reads.

For very large datasets where memory constraints are severe and some CPU overhead for deserialization is acceptable.

>--- **DISK_ONLY**

Stores data only on disk (serialized form).

Slowest storage level due to reliance on disk I/O.

Good for extremely large datasets that don't fit in memory, or for fault tolerance where data needs to be durable across executor restarts.

Significantly slower than memory-based storage levels.

When performance is less critical than fault tolerance or when datasets are too large for memory.

>--- **Replicated Storage Levels (e.g., MEMORY_ONLY_2, DISK_ONLY_2)**

These levels store two copies (2x replicated) of each partition across different nodes.

For example, MEMORY_ONLY_2 stores two copies in RAM on different executors.

Provides fault tolerance. If one executor or worker node goes down, the data can still be accessed from its replica, avoiding re-calculation from the DAG.

Doubles memory/disk consumption compared to non-replicated versions.

For highly critical data that is complex to calculate and must be readily available even if a node fails. Generally, cache() (which is MEMORY_AND_DISK) is preferred unless specific fault tolerance is required

>--- **Choosing the Right Storage Level**

| Storage Level        | Space Used | CPU Time | In Memory | On Disk | Serialized |
|---------------------|-----------|----------|-----------|---------|------------|
| MEMORY_ONLY         | High      | Low      | Yes       | No      | No         |
| MEMORY_ONLY_SER     | Low       | High     | Yes       | No      | Yes        |
| MEMORY_AND_DISK     | High      | Medium   | Partial   | Partial | No         |
| MEMORY_AND_DISK_SER | Low       | High     | Partial   | Partial | Yes        |
| DISK_ONLY           | Low       | High     | No        | Yes     | Yes        |

## **Salting**

Data Skew occurs when data is not distributed evenly across partitions. In a join operation, if one specific key (e.g., ID 1) has a massive number of records compared to others, all those records are sent to a single executor based on their hash.

The executor handling the skewed key becomes a bottleneck, taking significantly longer to finish while others sit idle, or it may even crash the application.

!!! Example
    A "best-selling product" might account for 90% of sales data, causing a massive skew for that specific product ID during a join.

> --- **Why Other Methods Might Fail**

- Re-partitioning: Standard re-partitioning often fails to solve skew because records with the same key are still hashed to the same partition.
- Broadcasting: While broadcasting the smaller table avoids shuffling, it is only viable if the table is small (e.g., <10MB). If both tables are large, broadcasting is not an option.
- AQE (Adaptive Query Execution): While Spark’s AQE provides some optimizations, it may not always resolve complex skew issues manually.

Salting involves adding a random value (the "salt") to the join key to break a single large partition into multiple smaller ones.

- Step 1: Modifying the Skewed Table (Left Table)

    In the skewed table, you append a random number (e.g., between 1 and 10) to the join key. This forces the records for the same ID to be distributed across 10 different partitions instead of one.

- Step 2: Modifying the Reference Table (Right Table)

    If you only salt the left table, the join will fail because the keys no longer match (e.g., "ID_1" vs "ID_1_5"). To fix this, you must replicate (explode) every record in the right table for every possible salt value used in the left table.

- Before Salting: A task might show a massive gap between the minimum duration (e.g., 0.2 seconds) and the maximum duration (e.g., 6 seconds) because one executor is struggling with the skewed partition.

- After Salting: The workload is balanced. The average time might be 9 seconds with a maximum of 12 seconds. While the total work might slightly increase due to replication, the overall job finishes faster because executors work in parallel rather than waiting for one skewed task to finish.

## **Dynamic Partition Pruning (DPP)**

Dynamic Partition Pruning (DPP) is an optimization technique in Apache Spark that enhances query performance, especially when dealing with partitioned data and join operations.

1. Understanding Partition Pruning

    Before diving into Dynamic Partition Pruning, it's essential to understand standard Partition Pruning.

    Partition pruning is a mechanism where Spark avoids reading unnecessary data partitions based on filter conditions. It "prunes" or removes data that is not relevant to the query.

    When data is partitioned on a specific column (e.g., sales_date), and a query applies a filter directly on that partitioning column, Spark can identify and read only the partitions that contain the relevant data. This significantly reduces the amount of data scanned.

    !!! Example
        - Imagine a large sales_data dataset partitioned by sales_date. Each date has its own partition.
        - If you run a query like SELECT  FROM sales_data WHERE sales_date = '2019-04-19', Spark, with partition pruning enabled, will only read the data for April 19, 2019.
        - Observed in Spark UI: In this example, if there are 123 total partitions, Spark will only read 1 file. The Spark UI's "SQL" tab details will show "Partition Filter" applied, indicating that the date has been cast and used for filtering.

2. The Issue: When Standard Partition Pruning Fails

    Standard partition pruning works efficiently when the filter condition is directly applied to the partitioning column of the table being queried. However, a common scenario where it fails is when:
    You have two dataframes (or tables), say df1 and df2.
    df1 is a partitioned table (e.g., partitioned by date).
    You need to join df1 and df2.
    The filter condition originates from df2 (the non-partitioned table or the table that is not the primary partitioned table being filtered).
    In such a case, because the filter is applied on df2 and not directly on df1's partitioning column, Spark's optimizer (without DPP) won't know which partitions of df1 to prune at the planning stage.

    !!! Example
        - Data: df1 is sales_data (partitioned by sales_date) and df2 is a date_dimension table (containing date and week_of_year columns).
        - Goal: Find sales data for a specific week, e.g., week = 16.
        - Query Concept: df1 is joined with df2 (e.g., on date columns), and then df2 is filtered for week_of_year = 16.
        - Configuration for Demonstration: To observe this issue, Spark's default behavior needs to be overridden by explicitly disabling Dynamic Partition Pruning (spark.sql.set('spark.sql.optimizer.dynamicPartitionPruning.enabled', 'false')) and also potentially disabling broadcast joins.
        - Observed in Spark UI: When this query is run with DPP disabled, Spark will scan all 123 files of the sales_data table, even though only a few dates (and thus partitions) might be relevant for week 16. The "Partition Filter" section in the Spark UI for df1 will show no effective pruning related to the join condition. This leads to performance degradation.

3. Dynamic Partition Pruning (DPP): The Solution

    Dynamic Partition Pruning (DPP) addresses the performance issue described above by enabling Spark to prune partitions at runtime.

    DPP is an optimization technique that allows Spark to update filter conditions dynamically at runtime.

    - Filter Small Table: Spark first filters the smaller table (df2, e.g., date_dimension for week = 16) to identify the relevant values (e.g., specific dates that fall in week 16).
    - Broadcast: The relevant values (e.g., the list of specific dates) from the filtered smaller table are broadcasted to all executor nodes. Broadcasting makes this small dataset available on all nodes where the larger table is processed.
    - Subquery Injection: At runtime, Spark then uses these broadcasted values to create a subquery (similar to an IN clause) for the partitioned table (df1). For instance, it essentially transforms the query to look like: SELECT  FROM big_table WHERE sales_date IN (SELECT dates FROM small_table).
    - Dynamic Pruning: This subquery allows Spark to dynamically identify and prune the irrelevant partitions of the large table (df1), reading only the necessary ones.

    !!! Example
        - Using the same sales_data (df1) and date_dimension (df2) tables, and the join with week = 16 filter.
        - Configuration: DPP is enabled (by default in Spark 3.0+ or explicitly enabled) and the broadcast mechanism is active.
        - Observed in Spark UI: When run with DPP enabled, Spark will only read a small subset of files (e.g., 3 files out of 123 total partitions), as only those files contain the dates relevant to week 16. The "Partition Filter" in the Spark UI will clearly show a "Dynamic Pruning Expression" applied to sales_date. You will also see "Broadcast Exchange" in the execution plan, indicating that the smaller table was broadcasted.

4. Key Conditions for Dynamic Partition Pruning

    For Dynamic Partition Pruning to work effectively, two primary conditions must be met:

    1. Partitioned Data: The data in the larger table (df1 in our example) must be partitioned on the column used in the join and filter condition (e.g., sales_date). If the data is not partitioned, DPP cannot apply.
    2. Broadcastable Second Table: The second table (df2), which provides the filter condition, must be broadcastable. This means it should be small enough to fit into memory and be efficiently broadcasted to all executor nodes. If it's too large, it won't be broadcasted, and DPP might not engage. You can also adjust Spark's broadcast threshold value if needed.

## **AQE**


AQE provides three main capabilities to improve performance:

1.  Dynamically Coalescing Shuffle Partitions.
2.  Dynamically Switching Join Strategies.
3.  Dynamically Optimizing Skew Joins.

> --- **Dynamically Coalescing Shuffle Partitions**

When Spark shuffles data (e.g., during a `groupBy` or `join`), it creates a default number of shuffle partitions—usually 200.

If the dataset is small, 200 partitions result in many empty or tiny partitions. This wastes resources because the Spark scheduler must still manage, schedule, and monitor tasks for these empty partitions, leading to unnecessary overhead.

AQE's Shuffle Reader observes the data size after the shuffle. If it finds many small partitions, it merges (coalesces) them into a smaller number of larger partitions at runtime.

!!! Example
    A 25 MB dataset originally split into 200 partitions might be coalesced by AQE into a single partition, reducing the number of tasks from 200 to 1. This saves CPU cores and scheduling time.

```python
# External Information: Enabling AQE and Coalescing
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
```

> --- **Dynamically Switching Join Strategies**

Spark usually decides a join strategy (like Sort-Merge Join or Broadcast Hash Join) during the initial planning phase based on the estimated size of the tables.

Initial estimates can be wrong. For example, a 10 GB table might be reduced to 5 MB after several filters and transformations. Without AQE, Spark would stick to the slower Sort-Merge Join.

AQE monitors the actual size of the data after transformations. If one side of the join becomes small enough (e.g., less than 10 MB), AQE dynamically switches the strategy from Sort-Merge Join to Broadcast Hash Join at runtime. Broadcast joins are significantly faster as they avoid the expensive shuffling and sorting required by Sort-Merge joins.

```python
# External Information: Conceptual Spark SQL example
# Initial tables are large (10GB and 20GB), triggering Sort-Merge Join
df1 = spark.table("fact_sales") # 10GB
df2 = spark.table("dim_products") # 20GB

# A filter is applied that reduces dim_products to 5MB
filtered_df2 = df2.filter(df2.category == "Electronics") 

# With AQE enabled, Spark switches to Broadcast Join at runtime 
# because filtered_df2 is now < 10MB.
result = df1.join(filtered_df2, "product_id")
```

> --- **Dynamically Optimizing Skew Joins**

Data skew occurs when data is distributed unevenly across partitions. For instance, if "Sugar" accounts for 80% of sales, the partition containing "Sugar" will be much larger than others.

One large partition causes a "199 out of 200 tasks completed" scenario, where one task takes a very long time or fails with an Out of Memory (OOM) error.

AQE identifies skewed partitions and splits them into smaller sub-partitions.
    1.  The large partition (e.g., Sugar data) is split into multiple smaller parts.
    2.  To ensure the join still works, the corresponding data on the other side of the join (the non-skewed table) is duplicated for each new sub-partition.
   Thresholds for Skew: AQE identifies a partition as skewed if:
    1.  The partition size is greater than 256 MB.
    2.  The partition size is more than 5 times the median partition size.

```python
# External Information: AQE Skew Join Configurations
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```