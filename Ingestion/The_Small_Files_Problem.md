### **The Small Files Problem (Streaming)**

This is the bridge between your real-time Kafka streams and your batch-oriented Data Lakehouse. It is also the number one reason cloud data bills explode and query engines choke.

#### **The Concept**

Streaming systems are designed to write data *frequently* to achieve low latency. If you dump Kafka topics to an S3 bucket (or MinIO) every 60 seconds so your dashboards are "real-time," you will generate 1,440 files per day, per partition, per table.

Object stores (like S3/MinIO) and query engines (like Trino) are optimized for streaming massive blocks of contiguous bytes. They want one 1GB file, not one hundred thousand 10KB files.

#### **The Brutally Honest Truth**

**"Real-time data lake" is an architectural oxymoron.** You have to cheat physics to make it work.

1. **The API Bankruptcy:** Cloud providers charge you per `PUT` and `GET` request. If you are writing tiny files every second, your S3 API costs will easily exceed your compute costs. You are paying to store metadata, not data.
2. **The "Header Tax":** When Trino executes a query, it has to open the file, read the Parquet magic bytes, read the footer, read the schema, and *then* read the data. If your file is 10KB, Trino spends 99% of its CPU time opening and closing files instead of processing data. Your blazing-fast distributed SQL engine becomes slower than a local Postgres instance.
3. **The NameNode Crash (Hadoop legacy):** If you are running on HDFS, every file takes up $\approx$ 150 bytes in the NameNode's RAM. Millions of small files will literally cause the master node to run out of memory and crash the entire cluster.

#### **Code/Action for `ai-data-orchestration**`

To solve this in a modern stack, you don't just write raw Parquet files anymore. You write to a table format like **Apache Iceberg**, which abstracts the file physics, and you configure your Flink writers to heavily buffer the data.

**File to add:** `storage/iceberg/flink_writer_config.sql`

When you define your Iceberg table in Flink SQL, you must manipulate the write properties to fight the small files problem at the source.

```sql
CREATE TABLE iceberg_catalog.ai_db.user_events (
    event_id STRING,
    user_id STRING,
    payload STRING,
    ts TIMESTAMP(3)
) PARTITIONED BY (DAYS(ts)) WITH (
    'connector' = 'iceberg',
    'catalog-name' = 'iceberg_catalog',
    'catalog-type' = 'rest',
    'uri' = 'http://rest-catalog:8181',
    
    -- THE MASTER SETTINGS
    -- -------------------
    
    -- 1. Target File Size
    -- Tell Flink to aim for 128MB files. Flink will keep data in memory/state 
    -- until it hits this threshold OR a checkpoint occurs.
    'write.target-file-size-bytes' = '134217728', 
    
    -- 2. Distribution Mode
    -- 'hash' prevents multiple Flink task managers from writing to the same 
    -- partition simultaneously, which creates a bunch of parallel small files.
    -- It shuffles data by partition key before writing.
    'write.distribution-mode' = 'hash',
    
    -- 3. The Unavoidable Reality: Compaction
    -- Even with the above, checkpoints force Flink to flush small files to disk.
    -- You MUST run a background compaction process. Iceberg allows you to do this
    -- without blocking readers.
    'format-version' = '2'
);

```

**Note**
Document that configuring the writer is only half the battle. You must add a cron job or an Airflow DAG that runs an Iceberg `RewriteDataFiles` action every night to bin-pack the tiny files created by Flink checkpoints into massive, optimized 1GB Parquet chunks for Trino to query the next day.

---
