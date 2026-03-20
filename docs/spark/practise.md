## **Spark Best Practices (Quick Revision)**

>---  **Handle Data Skew**

- Use salting
- Split skewed keys
- Increase partitions
- Prefer broadcast join for small tables

>--- **Avoid Unnecessary Shuffle**

- Use reduceByKey / aggregateByKey instead of groupByKey
- Use broadcast joins when possible

>--- **Memory Optimization**

- Avoid large collect()
- Cache only required data (persist wisely)
- Unpersist unused data

>--- **Partitioning**

- Use repartition (increase)
- Use coalesce (decrease)

>--- **Failure Handling**

- Driver failure → job fails
- Executor failure → tasks retry (default 4 times)

>--- **OOM Prevention**

- Don’t collect large data to driver
- Avoid large broadcasts
- Tune executor & driver memory

>--- **Resource Tuning**

- Tune:
  - `spark.executor.memory`
  - `spark.driver.memory`
  - `spark.sql.shuffle.partitions`
- Use dynamic allocation

## **Deciding the configuration**

The configuration of Spark executors, cores, and memory for processing large datasets like 100GB.

The primary takeaway is that there is no right or wrong "formula" to determine Spark configurations. Any solution depends on a trial-and-error approach combined with a deep understanding of your data and requirements.

### **Preliminary Requirement Gathering**

Before deciding on configurations, you must clarify the following with the interviewer or stakeholders:

- Data Type: Is the data semi-structured (JSON, XML), tabular (CSV), or optimized (Parquet)?.
- Transformations: Are there complex joins, wide dependencies, or just simple "select and dump" operations?.
- Data Sources: Is the data coming from a single source or multiple sources (e.g., 90GB from a file system and 10GB from a database)?.
- Service Level Agreement (SLA): How much time is allotted for the job to complete (e.g., 30 minutes vs. 2 hours)?.
- Cluster Configuration: What are the actual physical limits of the machines in the cluster?.

### **Understanding the Cluster Environment**

In the example provided, a real-world cluster used for these calculations consisted of:

16 Machines: 2 Master nodes (one for standby) and 14 Worker nodes.

Per Machine Specs: 128 GB RAM and 48-core CPUs.

Total Available Resources: Approximately 1.8 TB RAM and 672 cores across the 14 workers.

Multi-tenancy: These resources are shared among hundreds of projects and teams; one project cannot consume the entire cluster.

### **Configuration Strategy for 100GB+ Data**

>--- **Step 1: The "Starting Point" Mindset**

Large Data (>50GB): Start by allocating half the RAM of the total data volume. For 110GB of data, a starting point would be around 60GB of total RAM.

Small Data: For very small datasets (e.g., 1GB), you might actually need to multiply the RAM (e.g., 4-5GB) to account for overhead and execution.

>--- **Step 2: Executor and Core Allocation**

Cores per Executor: It is recommended to use 3 to 5 cores per executor. Using too many cores (e.g., 10+) can lead to excessive garbage collection (GC) and slow down the job.

RAM per Executor: A "decent" number to start with is 15GB per executor.
  
!!! Example
    
    Setup for ~110GB Data:
    
    Total RAM Needed: 60GB.
    
    Executors: 4.
    
    RAM per Executor: 15GB (4 executors  15GB = 60GB).
    
    Cores per Executor: 5.

>--- **Step 3: Driver Memory**
  
General Rule: Driver memory doesn't require as much complex calculation as executors because it primarily coordinates the tasks.
  
Initial Recommendation: For a 100GB dataset, start with 8GB to 10GB of driver memory. 

Warning: Never use `df.collect()` or `df.show()` on the full 100GB dataset, as this will pull all data to the driver and cause an Out of Memory (OOM) error.

>--- **Optimization via Spark UI**

After the first run, use the Spark UI to tune the performance:

Check for Data Skew: See if specific tasks are taking significantly longer than others.

Monitor Iterations: If the job is slow, check how many "iterations" or hydrations it takes to process the data with the current memory.

Adjusting for SLA: If the job takes 45 minutes but the SLA is 30 minutes, increase the number of executors or the RAM to reduce iterations.
