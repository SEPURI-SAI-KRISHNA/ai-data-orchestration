### Throttling & Rate Limiting (Token Bucket)

#### **The Concept**

Rate limiting isn't just about billing tiers; it's about system survival. The gold standard for this is the **Token Bucket** algorithm.

Imagine a literal bucket that holds a maximum of $N$ tokens.

1. Every time an API request comes in, it must take one token out of the bucket to proceed to Kafka.
2. If the bucket is empty, the request is instantly rejected (HTTP 429 Too Many Requests).
3. A background process (or mathematical formula) refills the bucket at a constant rate $R$ tokens per second, up to the maximum capacity $N$.

The formula applied at the moment of a request is:
$T_{current} = \min(T_{max}, T_{prev} + \Delta t \times R)$

This allows for **bursts** (up to $N$ requests instantly) while maintaining a strict long-term average rate ($R$).

#### **The Brutally Honest Truth**

**Most tutorials teach you rate limiting that actively destroys production systems.**

1. **The "In-Memory" Trap:** Developers often use an in-memory dictionary to track rate limits. When you deploy this in Docker/Kubernetes and scale out to 10 API pods, your rate limit is suddenly 10x higher than intended, and it's completely uneven depending on which pod the load balancer hits. **Distributed systems require distributed rate limiting.**
2. **The Redis Bottleneck:** To solve the above, you use Redis to store the token counts. But now, every single HTTP request requires a network hop to Redis *before* you can accept the data. If Redis is slow, your ingestion API slows down.
3. **The "Fail Open vs. Fail Closed" Dilemma:** What happens if Redis crashes?
* *Fail Closed:* Your API returns HTTP 500s because it can't check the rate limit. You just caused a total outage over a protective mechanism.
* *Fail Open:* You bypass the limit and accept the traffic. You risk overwhelming your downstream Kafka/DB cluster, but you don't drop data. (Hint: In data engineering, you usually want to *Fail Open* and rely on backpressure downstream).



#### **Code/Action for `ai-data-orchestration**`

Since a modern data orchestration stack often uses FastAPI for high-performance Python ingestion, we will write a distributed, Redis-backed Token Bucket algorithm that is safe for production.

**File to add:** `ingestion/api/rate_limiter.py`

```python
import time
import redis
from fastapi import HTTPException

# Assume redis_client is a global connection pool
# redis_client = redis.Redis(host='redis', port=6379, db=0)

class DistributedTokenBucket:
    def __init__(self, redis_client, key_prefix: str, max_burst: int, refill_rate_per_sec: float):
        self.redis = redis_client
        self.prefix = key_prefix
        self.max_burst = max_burst
        self.refill_rate = refill_rate_per_sec

    def consume(self, client_id: str, tokens: int = 1) -> bool:
        """
        Uses a Lua script in Redis to ensure atomic execution. 
        If you do this in Python (GET then SET), you will have massive race conditions.
        """
        lua_script = """
        local key = KEYS[1]
        local tokens_requested = tonumber(ARGV[1])
        local max_burst = tonumber(ARGV[2])
        local refill_rate = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])

        local bucket = redis.call('HMGET', key, 'tokens', 'last_update')
        local current_tokens = tonumber(bucket[1]) or max_burst
        local last_update = tonumber(bucket[2]) or now

        -- Calculate refilled tokens
        local delta_time = math.max(0, now - last_update)
        local refilled = delta_time * refill_rate
        current_tokens = math.min(max_burst, current_tokens + refilled)

        if current_tokens >= tokens_requested then
            -- Allow request
            current_tokens = current_tokens - tokens_requested
            redis.call('HMSET', key, 'tokens', current_tokens, 'last_update', now)
            -- Set TTL to prevent Redis memory bloat for inactive clients
            redis.call('EXPIRE', key, math.ceil(max_burst / refill_rate) * 2)
            return 1
        else:
            -- Reject request
            redis.call('HMSET', key, 'tokens', current_tokens, 'last_update', now)
            return 0
        end
        """
        try:
            # We execute the script atomically in Redis
            result = self.redis.eval(
                lua_script, 1, f"{self.prefix}:{client_id}", 
                tokens, self.max_burst, self.refill_rate, time.time()
            )
            return bool(result)
        except redis.RedisError as e:
            # BRUTAL TRUTH: Fail Open! If Redis is down, we do not want to block ingestion.
            # Log the error heavily, but let the data through to Kafka.
            print(f"Redis rate limiter failed, failing open: {e}")
            return True

# Usage in FastAPI Dependency:
# if not bucket.consume(client_ip):
#     raise HTTPException(status_code=429, detail="Too Many Requests")

```

**Note**
Document why we use a **Lua Script**. If you read the token count, do the math in Python, and write it back, concurrent requests from the same user will overwrite each other, allowing them to bypass the limit entirely. Lua scripts in Redis run atomically.
