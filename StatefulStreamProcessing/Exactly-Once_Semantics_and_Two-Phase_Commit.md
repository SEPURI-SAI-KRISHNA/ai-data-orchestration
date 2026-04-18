### **Exactly-Once Semantics (EOS) & Two-Phase Commit (2PC)**

#### **The Concept**

If a server's power cord is ripped out while it is calculating a user's bank balance, what happens when the server reboots?

* **At-Most-Once:** The system drops the message. The bank balance is too low. (Unacceptable).
* **At-Least-Once:** The system retries the message. The bank balance is updated twice. (Also unacceptable).
* **Exactly-Once (EOS):** The system guarantees that the final state is as if the failure never occurred. The math is perfectly correct.

To achieve this, engines like Flink use **Checkpoint Barriers**. Flink injects a special "barrier" marker into the Kafka stream. When an operator (like a window function) receives this barrier, it stops processing, takes a snapshot of its current memory (state), saves it to reliable storage (like S3), and then passes the barrier downstream.

To make this work *end-to-end* (from Kafka, through Flink, back to Kafka or Iceberg), the sink must support a **Two-Phase Commit (2PC)**:

1. **Pre-Commit:** Flink writes the data to the destination, but tells the destination "keep this hidden for now."
2. **Commit:** Once Flink verifies that the snapshot was successfully saved to S3, it tells the destination "make the data visible."

#### **The Brutally Honest Truth**

**"Exactly-Once" is a lie. The correct term is "Effectively-Once processing."** 1.  **The Consumer Illusion:** Data *will* be replayed during a recovery. Flink will re-read the same Kafka messages. The magic is that Flink rolls back its internal state to the exact moment before those messages were processed, so the *result* is exactly once. If your Flink job makes an external HTTP API call to a non-transactional system during processing, that API *will* be hit twice. EOS does not protect the outside world.
2.  **The Kafka "Hanging Transaction" Trap:** If you use 2PC to write from Flink back to Kafka, and the Flink job crashes *after* the Pre-Commit but *before* the Commit, you have a hanging transaction in Kafka. If your downstream Kafka consumers are set to `isolation.level=read_committed`, **they will permanently stop reading data** because they are blocked waiting for that transaction to resolve.
3.  **The Latency Cost:** 2PC means your downstream systems (like Trino reading Iceberg) will not see the data until the checkpoint completes. If your checkpoint interval is 5 minutes, your "real-time" streaming pipeline suddenly has a hard 5-minute latency floor.

#### **Code/Action for `ai-data-orchestration**`

We must configure Flink to use EXACTLY_ONCE, but we must tune the transaction timeouts to prevent the pipeline from permanently hanging during a crash.

**File to add:** `processing/flink/checkpointing_config.yaml`

```yaml
# THE MASTER SETTINGS
# -------------------

# 1. Enable Checkpointing
execution.checkpointing.interval: 60000 # 1 minute

# 2. The Semantic Guarantee
execution.checkpointing.mode: EXACTLY_ONCE

# 3. Prevent the "Death Spiral"
# If a checkpoint takes 2 minutes, and your interval is 1 minute,
# Flink will spend 100% of its CPU taking checkpoints and 0% processing data.
# This forces Flink to wait at least 15 seconds between checkpoints.
execution.checkpointing.min-pause: 15000

# 4. Timeout configuration
execution.checkpointing.timeout: 600000 # 10 minutes

# --- KAFKA SINK CONFIGURATION (Crucial) ---
# When Flink writes to Kafka using EXACTLY_ONCE, it uses Kafka Transactions.
# The default Kafka transaction timeout is 15 minutes. 
# If Flink is down for > 15 minutes, Kafka aborts the transaction. 
# When Flink wakes up, it will try to commit an aborted transaction and crash loop forever.
# YOU MUST align these timeouts. Set this in your Flink Kafka Sink properties:
transaction.timeout.ms: 900000 # 15 minutes

```

**Note**
If you are writing to **Apache Iceberg**, Iceberg supports exactly-once inherently through its optimistic concurrency control and snapshot isolation. Flink will only commit the Iceberg manifest file upon a successful checkpoint. Document this integration as the primary reason for choosing Iceberg over raw Parquet files for your data lake.

---
