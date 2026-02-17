# The Concept

In a distributed system, the Producer (App) and the Consumer (Spark/Flink) are often built by different teams. If the App team changes the data structure (e.g., renames `user_id` to `uuid`), the Consumer crashes.

To prevent this, we introduce a **Schema Registry**. It acts as a central authority.

1. **Producer:** "Hey Registry, I want to send data with this shape (Schema A). Is that allowed?"
2. **Registry:** "Yes, Schema A is compatible with previous versions. Here is ID #55."
3. **Producer:** Sends data to Kafka, but instead of the full schema, it just sends `ID #55` + the binary data (Avro/Protobuf).
4. **Consumer:** Reads `ID #55`, asks the Registry "What is #55?", gets Schema A, and deserializes the data.

## The Brutally Honest Truth

**JSON is technical debt.**

1. **"Schemaless" is a Lie:** There is always a schema. It’s just implicitly defined by the *last* person who touched the code. If you use JSON in Kafka, you are sending the field names (`"first_name": "..."`) in *every single message*. You are wasting 40-60% of your bandwidth transmitting redundant strings.
2. **The Compatibility Hell:**
* **Backward Compatibility:** (Required for consumers) The new schema can read old data. This means you **cannot delete mandatory fields**.
* **Forward Compatibility:** (Required for producers) The old schema can read new data. This means you **cannot add mandatory fields**.
* **Full Compatibility:** You can only add optional fields and delete optional fields.


3. **The "Default" Trap:** If you add a new field to your Avro schema, you **MUST** provide a `default` value. If you don't, your old consumers (who don't know about this field) will crash when they try to read the new data, or your new consumers will crash reading old data.

## Code/Action for ai-data-orchestration

We will enforce **Avro** with **Full Transitive Compatibility**.

**File to add:** `schemas/avro/user_created_v1.avsc`

```json
{
  "type": "record",
  "name": "UserCreated",
  "namespace": "com.ai_orchestration.events",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "timestamp", "type": "long"},
    {"name": "user_id", "type": "string"},
    {"name": "email", "type": "string"},
    
    // THE MASTER MOVE:
    // Always make fields nullable (union with null) if you think
    // they might ever be deprecated.
    // AND provide a default value.
    {
      "name": "referral_code",
      "type": ["null", "string"],
      "default": null
    }
  ]
}

```

**Configuration Note for the Repo:**
In your Kafka Producer configuration (e.g., Python/Java), you must point to the registry and enable the serializer.

```properties
# producer.properties
key.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
value.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
schema.registry.url=http://schema-registry:8081

# AUTO-REGISTRATION IS FOR DEV ONLY
# In PROD, you disable this. You register schemas via CI/CD pipelines 
# using the Schema Registry Maven Plugin or API.
auto.register.schemas=false

```

**Next Step:**
We have the structure. Now we need to guarantee that data arrives exactly once, even if the network fails 50 times.
