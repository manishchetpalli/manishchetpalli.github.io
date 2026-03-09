## **Introduction**

Develop a platform tailored for Registered Investment Advisors (RIAs) to delve into key performance indicators (KPIs). The platform will be engineered to centralize and scrutinize financial information from numerous clients, each presenting their distinct data sources and formats.

Given the diverse nature of our clients and their data, our Data Lakehouse must possess the capability to manage a broad spectrum of financial entities.

These encompass accounts, transactions, advisors, and assets, each characterized by unique attributes:

1. Accounts encompass individual or collective holdings, housing a moderate volume of data and relatively stable information.

2. Transactions represent the dynamic financial activities within accounts, characterized by a high volume and frequent updates.

3. Advisors furnish details about financial advisors, establishing connections to their managed accounts and transactions. Although low in volume, this data is pivotal for connectivity.

4. Assets encompass the array of investments within accounts, exhibiting moderate volume and variability.

This assortment, coupled with the necessity to accommodate diverse clients and data formats, underscores the importance of a meticulous approach to architecting our Data Lakehouse. It is imperative that the architecture facilitates efficient data ingestion, storage, and analysis across all client datasets.

## **Design Questions**

1. Directory Hierarchy and Multi-Tenancy

	How would you design the directory hierarchy in our Data Lakehouse to efficiently manage and isolate data from multiple clients, considering the diversity of data sources and formats?

    Separate buckets per client (strong isolation)

    Single bucket with tenant_id partition (cost efficient)

    For financial data: Logical isolation + row-level security + encryption per tenant

    ```
        /lakehouse
            /client_a
                /bronze
                    /accounts
                    /transactions
                    /advisors
                    /assets
                /silver
                /gold
            /client_b
                /bronze
                /silver
                /gold
    ```

2. Data Partitioning and Format Considerations

	Given the high volume of transactions compared to other entities, what partitioning strategy would you employ to enhance query performance and data accessibility? 

    How would your approach facilitate analysis across varying data formats and sources?

    ```
        transactions/
            year=2026/
                month=03/
                    day=04/
    ```

    If scale grows:

    - Partition by year/month

    - Z-Order / Cluster by account_id

    - OR bucket by account_id

3. ETL Pipeline Design

    What key considerations would you prioritize in designing an ETL pipeline for our solution, ensuring scalability and adaptability to different client data formats and structures?

	How would you approach the challenge of transforming diverse data formats into a structured format that supports efficient analysis and KPI generation?

    Key Consiferations:
    
    - Schema Variability: Schema registry/Config driven pipelines
    - Scalability : Spark/Flink
    - Idempotencies: Merge/CDC/Watermarking
    ```
    Client Sources
        ↓
    Landing Zone (Raw Storage)
        ↓
    Ingestion Layer (Spark/Flink)
        ↓
    Bronze (Raw, versioned)
        ↓
    Transformation Layer
        ↓
    Silver (Cleaned & Standardized)
        ↓
    Aggregation Layer
        ↓
    Gold (KPI Tables)
    ```

4. Adapting to Data Evolution

    Considering the financial sector’s dynamic nature, how would you ensure that the Data Lakehouse and ETL processes remain flexible to accommodate new types of data entities and evolving relationships over time?

    - Schema Evolution Support: Delta/Iceberg
    - Metadata Driven Architecture: dynamic transformations/store schema metadata in tables
    - Versioned gold tables
    - Decoupled pipelines: ingestion independent of transformation
    - Normalization: reduce redundancy and improve data integrity

## **Extra Questions**

1. Is Higher Normal Form Better or Worse for Transactional DB?

    - For OLTP:

        Higher normalization (3NF) → Better
        Avoids anomalies
        Reduces redundancy
        Ensures integrity

    - For OLAP:

        Too much normalization → Slower joins

        So OLTP → 3NF preferred. OLAP → Denormalized/star schema preferred

2. Is Snowflake Schema Suited For Transactional DB or Reporting DB?

    Snowflake Schema is suited for: Reporting / OLAP DB Not transactional.

    Because: Multiple joins, Optimized for analytics ,Normalized dimensions.

3. One Big Table (OBT Approach)

    Commonly used in analytics.

    - Advantages: Faster queries (no joins), Simple dashboard logic, Better BI performance, Good for KPI reporting

    -  Disadvantages: Data duplication, Storage heavy, Harder updates, Schema evolution complex, Not suitable for OLTP.