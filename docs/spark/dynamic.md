## **AQE**

> --- **Key Features of AQE**

AQE provides three main capabilities to improve performance:
1.  Dynamically Coalescing Shuffle Partitions.
2.  Dynamically Switching Join Strategies.
3.  Dynamically Optimizing Skew Joins.

## **Dynamically Coalescing Shuffle Partitions**

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

## **Dynamically Switching Join Strategies**

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

## **Dynamically Optimizing Skew Joins**

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
