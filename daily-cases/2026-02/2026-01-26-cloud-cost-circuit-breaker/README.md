# 📅 Daily Case: Stopping a Cloud Cost Leak (Infinite Loops)

**Date:** 2026-01-26  
**Tags:** #AWSLambda #DynamoDB #CostManagement #CircuitBreaker #Redis #Reliability

## 1. Problem Statement
A deployment error created a **recursive feedback loop** in our architecture.
1.  **Event:** Lambda writes to DynamoDB.
2.  **Trigger:** DynamoDB Streams triggers the Lambda again on "Update".
3.  **Loop:** The Lambda updates the "LastModified" timestamp, creating a new stream event, triggering itself infinitely.

**Impact:** The system scaled to 10,000 invocations/minute, increasing the AWS bill by **$5,000** in a few hours due to unmanaged compute resources.

## 2. Business & Technical Assumptions
- **Latency:** Real-time inventory updates are critical, but system stability is higher priority.
- **Budget:** We must prevent "Bill Shock" ($5k spike is unacceptable).
- **Recovery:** We need a way to stop the bleeding immediately without waiting for a code deployment pipeline (CI/CD takes 20 mins).

## 3. Engineering Analysis
I analyzed three stages of intervention: **Emergency**, **Prevention**, and **Protection**.

### Stage 1: The "Kill Switch" (Immediate Action)
- **Problem:** How to stop the cost spike *now*?
- **Solution:** Set AWS Lambda **Reserved Concurrency to 0**.
- **Why:** This immediately throttles all incoming requests at the AWS infrastructure level. It is faster and safer than deploying a hotfix.

### Stage 2: Architectural Decoupling (Prevention)
- **Problem:** Direct triggers (`DB -> Lambda`) are dangerous for cost control.
- **Solution:** Introduce **AWS SQS** between the Event Source and the Processor.
- **Benefit:** If a loop occurs, the queue fills up (cheap storage) instead of spinning up thousands of Lambda instances (expensive compute). This provides **Cost Decoupling**.

### Stage 3: Circuit Breaker Pattern (Protection - Chosen Strategy)
- **Problem:** How to detect logical loops automatically in the future?
- **Solution:** Implement a **Rate Limiting / Circuit Breaker** logic using **Redis**.
- **Logic:** If a specific `ProductID` is updated more than 10 times in 1 minute, the system identifies it as an anomaly and stops processing that specific item.



## 4. Solution Design
**New Architecture:** `CDC Event -> SQS -> Lambda (Worker) <-> Redis (State Store) -> DynamoDB`

1.  **Input:** SQS buffers the traffic burst.
2.  **Check:** Lambda checks Redis: *"Has this ItemID been processed > 10 times in 60 seconds?"*
3.  **Act:**
    - **If NO:** Process data and increment Redis counter.
    - **If YES:** Discard message (Circuit Open), log error, and alert via Slack.

## 5. Minimal Working Example (The Logic)

```python
import redis

# Connect to Redis (Our "Short-term Memory")
r = redis.Redis(host='localhost', port=6379, db=0)

def process_event(event):
    product_id = event['product_id']
    
    # 1. Check Rate Limit (Circuit Breaker)
    # Key strategy: "counter:product_123"
    key = f"counter:{product_id}"
    
    current_count = r.get(key)
    
    if current_count and int(current_count) > 10:
        print(f"Circuit Breaker Open! Loop detected for {product_id}")
        # In a real scenario: Send alert to Slack/PagerDuty here
        return # STOP PROCESSING
        
    # 2. Business Logic (Safe to proceed)
    update_dynamodb(product_id)
    
    # 3. Increment Counter & Set Expiry (TTL = 60 seconds)
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, 60) # Auto-reset after 1 minute
    pipe.execute()
```

## 6. Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Redis Goes Down** | Circuit breaker fails (fail-open or fail-closed). | Wrap Redis calls in `try-catch`. If Redis fails, log the error but allow processing (**Fail-Open**) to avoid stopping business, or block (**Fail-Closed**) if cost risk is extreme. |
| **SQS Fills Up** | Storage cost increases slightly. | Set a `retention period` (e.g., 1 day) and configure a **Dead Letter Queue (DLQ)** for unprocessable messages. |
| **False Positive** | Valid heavy traffic is blocked. | Tune the threshold (e.g., increase limit from 10 to 50) based on real-world usage patterns observed in CloudWatch. |
