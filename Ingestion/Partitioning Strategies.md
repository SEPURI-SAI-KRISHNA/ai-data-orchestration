
This topic separates systems that scale gracefully from systems that wake you up at 3:00 AM because a single server caught on fire while the rest of the cluster did nothing.

# **The Concept**

In distributed logs like Kafka (or databases like Cassandra), you cannot store all data on one machine. You must split the topic/table into **Partitions**.

When a Producer sends a message, it has to decide *which* partition gets the data.

1. **No Key (Round Robin/Sticky):** The Producer just distributes messages evenly across all partitions. Excellent for throughput, but you lose all guarantees of chronological order.
2. **Keyed (Semantic Hash):** You provide a key (e.g., `user_id`). The system runs a formula: `hash(user_id) % num_partitions = partition_number`. This guarantees that all events for a specific user go to the exact same partition, in the exact order they occurred.

## **The Brutally Honest Truth**

**Default partitioning algorithms will eventually destroy your cluster.**

1. **The "Justin Bieber" Problem (Data Skew):** Let's say you partition by `tenant_id`. Tenant A is a mom-and-pop shop with 10 events a day. Tenant B is a massive enterprise generating 10,000 events a second. Because of the hash formula, *all* of Tenant B's traffic is forcefully routed to a single partition, residing on a single disk, processed by a single consumer thread. That broker will max out its CPU and crash, while your other brokers sit at 2% utilization.
2. **The Modulo Trap:** You start with 10 partitions. You realize you need more throughput, so you add 5 more partitions. Suddenly, `hash("user_123") % 10 = 4`, but `hash("user_123") % 15 = 12`. Your new events for this user are now going to a different partition than their historical events. **You just permanently destroyed the strict ordering of your data.**
3. **Sticky Partitions vs. True Round Robin:** Modern Kafka producers don't actually do true event-by-event round-robin by default anymore. To save network overhead, they "stick" to one partition for a few milliseconds, dump a batch, and then move to the next. If your batch sizes are misconfigured, you can accidentally create micro-hot-spots.

## Code/Action for `ai-data-orchestration`

To fix the Data Skew problem, architects use a technique called **Salting**.

**File to add:** `ingestion/partitioning/salted_partitioner.py`

If a key is too "hot," we artificially split it by appending a random number (the salt) to the key, spreading the load across a defined subset of partitions.

```python
import hashlib
import random

class SaltedPartitioner:
    def __init__(self, num_partitions, hot_keys, salt_range=3):
        self.num_partitions = num_partitions
        self.hot_keys = hot_keys # e.g., ['enterprise_tenant_xyz']
        self.salt_range = salt_range # Spread hot keys across 3 partitions

    def partition(self, key, value):
        if key in self.hot_keys:
            # It's a massive tenant. We sacrifice strict global ordering 
            # for this tenant to keep the cluster alive. 
            # We append a random salt (0, 1, or 2) to the key.
            salt = random.randint(0, self.salt_range - 1)
            salted_key = f"{key}_{salt}"
            return self._hash(salted_key)
        else:
            # Normal tenant. Keep strict ordering.
            return self._hash(key)

    def _hash(self, key):
        # standard murmur2 or md5 hash modulo num_partitions
        hash_val = int(hashlib.md5(key.encode('utf-8')).hexdigest(), 16)
        return hash_val % self.num_partitions

```

**What to read further**

* **The Trade-off:** Explain that salting a key means you now have to do a "Scatter-Gather" read on the consumer side if you need to reconstruct the exact timeline of that specific enterprise tenant.
* **Over-partitioning:** Why creating 1,000 partitions for a topic doing 10 messages a second is a terrible idea (it exhausts ZooKeeper/KRaft metadata and increases latency).
