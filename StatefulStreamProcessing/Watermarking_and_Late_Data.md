### **Watermarking & Late Data**

Watermarks are the mathematical compromise you make to deal with that mess.

#### **The Concept**

A Watermark is a hidden metadata record flowing through your stream that declares: **"I promise that no more events with a timestamp older than $T$ will ever arrive."**

When Flink is calculating a 1-minute window (e.g., 10:00 to 10:01), it keeps that window open in memory. It only computes the final result and closes the window when it receives a Watermark $\ge$ 10:01.

But what happens if an event from 10:00:45 arrives at 10:05:00, long after the Watermark has passed? That is **Late Data**.

#### **The Brutally Honest Truth**

**Watermarks are always a guess, and the default behavior for late data is to silently delete it.**

1. **The "Slowest Partition" Bottleneck:** Watermarks propagate through your pipeline by taking the *minimum* value across all parallel Kafka partitions. If you have 10 partitions and one of them stops receiving data (or its producer crashes), its watermark never advances. **Your entire Flink job will freeze.** The windows will stay open forever, your RAM will fill up, and no output will be produced. (You must configure `idleness` detection to fix this).
2. **The Latency vs. Correctness Trade-off:** * If you set your Watermark delay to 1 second: Your pipeline is blazing fast, but you will drop 5% of your events because they arrive 2 seconds late.
* If you set your delay to 1 hour: Your data is 100% perfectly accurate, but your "real-time" dashboard is now delayed by an hour.


3. **The "Silent Drop" Tragedy:** By default, if data arrives after the watermark has passed and the window is closed, Flink drops it on the floor. No error is thrown. No log is written. You just lose money or metrics.

#### **Code/Action for `ai-data-orchestration**`

To build a professional orchestration system, you must never silently drop data. You must catch late data and route it somewhere else (like a database table for offline reconciliation) or explicitly allow the window to update.

**File to add:** `processing/flink/jobs/late_data_handler.py` (Using PyFlink)

```python
from pyflink.datastream import StreamExecutionEnvironment, OutputTag
from pyflink.datastream.window import TumblingEventTimeWindows
from pyflink.common.time import Time

env = StreamExecutionEnvironment.get_execution_environment()

# THE MASTER SETTING: The Side Output Tag
# We create a special tag to catch the "trash" that Flink tries to throw away.
late_data_tag = OutputTag("late-data")

# Assume 'stream' is your pre-configured Kafka source with Watermarks assigned
# stream = ... 

windowed_stream = stream \
    .key_by(lambda x: x['user_id']) \
    .window(TumblingEventTimeWindows.of(Time.minutes(1))) \
    .allowed_lateness(Time.minutes(5)) \
    .side_output_late_data(late_data_tag) \
    .process(YourCustomWindowFunction())

# 1. The Main Output: This goes to your real-time dashboard (Trino/Superset)
main_results = windowed_stream

# 2. The Late Output: This goes to an Iceberg "reconciliation" table
# These are the events that missed the 5-minute allowed lateness window.
late_results = windowed_stream.get_side_output(late_data_tag)

# You now sink 'late_results' to a separate storage path so analysts 
# can manually adjust end-of-day financial reports.

```

**Note**

* **Allowed Lateness:** Notice the `allowed_lateness(Time.minutes(5))` setting. This tells Flink to emit the main window result, but keep the window state in memory for another 5 minutes. If late data arrives during that 5 minutes, Flink will emit an *update* (a retraction and a new positive record). Your downstream system (like Iceberg) must support `UPDATE` operations to handle this.
* **Idleness Timeout:** Document the `watermark_strategy.with_idleness(...)` setting. This is mandatory in production to prevent a single quiet Kafka partition from stalling the entire multi-node cluster.

---
