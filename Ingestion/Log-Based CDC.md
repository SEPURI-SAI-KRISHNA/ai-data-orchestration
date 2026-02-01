# The Concept
Instead of querying the database (which puts load on the engine and misses deletes), we act like a "replica." We read the Write-Ahead Log (WAL) (Postgres) or Binary Log (Binlog) (MySQL). These logs record every single byte change to the disk. We stream these changes to Kafka.

## The Brutally Honest Truth
CDC is sold as the "magic bullet" for real-time data. In reality, CDC is a fragile tether to a database that hates you.

1. The "Slot" Problem: In Postgres, you use a "Replication Slot" to track your position in the WAL. If your CDC consumer (Debezium) goes down, the Postgres database will not delete old WAL files. It holds them for you, hoping you come back.

    - The Result: Your production database runs out of disk space and crashes. The DBA screams at you.

2. The "Snapshot" Lock: When you first start, you need a snapshot of the existing data. This often requires a read-lock or holds long-running transactions open, preventing vacuuming (cleanup) processes on the DB. You can freeze a production app just by trying to initialize your pipeline.

3. Schema Evolution breaks everything: If a developer renames a column, the WAL keeps emitting binary changes. Your parser (Debezium) might crash because the registry doesn't match the binary stream.

4. Toast Columns: Large text/blob fields in databases are often stored "off-row" (TOAST in Postgres). If an update happens to another column, the WAL might not contain the TOAST data (to save space). Your stream receives a null for a value that isn't actually null, it just wasn't changed.

## Code/Action for ai-data-orchestration

You need to add a Debezium connector configuration, but you must add the safety valves that tutorials ignore.


File to add: ingestion/debezium-postgres-connector.json


```
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "postgres",
    "database.port": "5432",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_ai_slot",
    
    // THE MASTER SETTINGS (The "Mysterious" part)
    
    // 1. Prevent the DB crash: Drop the slot if we are too far behind?
    // Actually, Debezium can't do this easily, you monitor this via heartbeats.
    "heartbeat.interval.ms": "5000",
    
    // 2. Handle the "TOAST" problem (Replica Identity)
    // You must run: ALTER TABLE x REPLICA IDENTITY FULL; on the DB side 
    // or Debezium will send NULLs for unchanged columns.
    
    // 3. Snapshot Mode: 'initial' is standard, but 'never' is safer if 
    // you are connecting to a critical Prod DB and will backfill later.
    "snapshot.mode": "initial",
    
    // 4. Decimal Handling: Java implementation of Decimals is precise
    // but Spark/Python hates it. Convert to Double? No, use String to be safe.
    "decimal.handling.mode": "string",
    
    // 5. Tombstones: When a record is deleted, Debezium sends a NULL value 
    // to mark the key as gone. Kafka needs this for Log Compaction.
    "tombstones.on.delete": "true"
  }
}
```

__What to study ?__:

- __Why REPLICA IDENTITY FULL is required__ for your specific AI use cases (you need the full row for retraining, not just the diff).

- __Heartbeat tables__: How to use a dummy write to the DB to ensure the WAL position advances even if traffic is low.