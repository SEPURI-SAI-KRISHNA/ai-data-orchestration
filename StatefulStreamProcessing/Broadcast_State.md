### **Broadcast State**

We know how joining two massive streams is a mathematical nightmare. But what if one of the streams isn't massive? What if you need to push a new AI routing rule or a dynamic ML threshold to 1,000 parallel workers instantly, without restarting the cluster?

#### **The Concept**

Imagine you are processing 1 million credit card swipes a second across 100 Flink TaskManagers. Suddenly, your data science team detects a new fraud pattern and updates a config database: "Block all transactions over $500 from Merchant ID 999."

You cannot have your 100 Flink workers query a Postgres database for every single swipe. You would DDOS your own database instantly.

Instead, you use **Broadcast State**. You ingest the config changes as a slow-moving stream (e.g., via Debezium CDC from Topic 1), and Flink literally broadcasts a copy of every single rule to the local memory of *every single parallel worker*. The workers then evaluate the high-volume stream against their local, in-memory copy of the rules. Zero network calls. Zero database latency.

#### **The Brutally Honest Truth**

**Broadcast State is the most abused feature in stream processing. It is not a distributed database.**

1. **The OOM Cluster Nuke:** Developers often think, "Wow, I can just broadcast my `Users` table so I can join `user_id` to `email`!" If your user table is 50GB, and you have 20 TaskManagers, you just consumed 1 Terabyte of cluster RAM holding redundant copies of the same data. Your cluster will immediately Out-Of-Memory (OOM) and die. Broadcast state is for *megabytes* of data (rules, thresholds, feature flags), not gigabytes of dimension data.
2. **The Race Condition:** Flink processes the main stream and the broadcast stream asynchronously. If a user changes their preference in the UI (Broadcast Stream) and immediately clicks a button (Main Stream), the button click might reach the Flink operator *before* the new preference does. Your system will process the event using the old rule. There is no strict temporal synchronization between the two streams.
3. **The Read-Only Trap:** The main stream can *read* the broadcast state, but it cannot *write* to it. If you need a state object that both streams update collaboratively, Broadcast State is the wrong pattern.

#### **Code/Action for `ai-data-orchestration**`

We will implement a dynamic ML threshold router using PyFlink. This pattern separates the high-volume inference stream from the low-volume model configuration stream.

**File to add:** `processing/flink/jobs/dynamic_threshold_router.py`

```python
from pyflink.datastream import MapStateDescriptor
from pyflink.datastream.functions import CoBroadcastProcessFunction
from pyflink.common.typeinfo import Types

# THE MASTER SETTING: The State Descriptor
# This defines the schema of the dictionary that will be broadcasted to all workers.
# Key: Rule ID (String), Value: Threshold (Float)
threshold_state_desc = MapStateDescriptor(
    "dynamic-thresholds", 
    Types.STRING(), 
    Types.FLOAT()
)

class FraudEvaluationFunction(CoBroadcastProcessFunction):
    
    def process_element(self, transaction, read_only_ctx, out):
        # 1. This handles the massive, high-volume transaction stream.
        # We access the broadcast state in READ-ONLY mode.
        state = read_only_ctx.get_broadcast_state(threshold_state_desc)
        
        merchant_id = transaction['merchant_id']
        amount = transaction['amount']
        
        # Default to $10,000 if no rule exists, otherwise use the dynamic rule
        max_allowed = state.get(merchant_id) if state.contains(merchant_id) else 10000.0
        
        if amount > max_allowed:
            out.collect(f"FRAUD ALERT: {transaction['id']}")
        else:
            out.collect(f"CLEARED: {transaction['id']}")

    def process_broadcast_element(self, rule, ctx, out):
        # 2. This handles the slow-moving configuration stream.
        # This is the ONLY place where you can write to the Broadcast State.
        state = ctx.get_broadcast_state(threshold_state_desc)
        
        # Update the rule in local memory for this specific worker
        state.put(rule['merchant_id'], rule['max_amount'])
        print(f"Worker updated rule for Merchant {rule['merchant_id']}")

# Connecting the streams in the main job graph:
# main_stream.connect(rule_stream.broadcast(threshold_state_desc)) \
#            .process(FraudEvaluationFunction())

```

**Note**
Document that Broadcast State is stored on the Java Heap, not in RocksDB. Because it is small, keeping it on the heap makes access virtually instantaneous, completely eliminating the serialization tax we discussed in Topic 14.

---
