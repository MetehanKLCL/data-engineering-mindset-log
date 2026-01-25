# 📅 Daily Case: Handling High Concurrency Traffic Spikes

**Date:** 2026-01-25  
**Tags:** #SystemDesign #Decoupling #Idempotency #CloudCost #TradeOffs

---

## 1. Problem Statement

The **Smart Drug Dispenser** system experiences a massive **100x traffic spike** between **08:00 – 09:00 AM** due to synchronized hospital medication rounds.

During this peak window, thousands of IoT devices attempt to write logs simultaneously, leading to:

- **Database Connection Timeouts**  
  PostgreSQL reaches the `max_connections` limit.
- **High Latency**  
  Dashboard users experience frozen or delayed UI updates.
- **Inefficient Resource Usage**  
  The database must be over-provisioned to handle a one-hour daily peak.

---

## 2. Business & Technical Assumptions

- **Budget Constraint**  
  We cannot afford a high-capacity database instance that remains idle for ~95% of the day.

- **Latency Tolerance**  
  A delay of **1–5 minutes** for log ingestion during peak hours is acceptable.

- **Data Integrity**  
  **Zero data loss** is permitted.

- **Scale Expectation**  
  Approximately **100,000 write requests within one hour** during peak traffic.

---

## 3. Engineering Analysis & Trade-offs

I evaluated multiple architectural strategies with a strong focus on **cost efficiency**, **scalability**, and **operational simplicity**.

---

### Strategy A: Vertical Scaling (Upsizing the Database)

**Approach:**  
Upgrade the PostgreSQL instance to handle peak concurrency directly.

**Cost Comparison:**

| Instance Type | Monthly Cost |
|--------------|--------------|
| db.t3.medium | ~$60         |
| db.r5.2xlarge | ~$1000       |

**Observation:**  
This represents a **~16x cost increase** to support a load that occurs for only **one hour per day**.

**Verdict:**  Rejected  
Poor ROI due to high fixed cost for infrequent peak usage.

---

### Strategy B: Decoupling with a Queue (Selected)

**Approach:**  
Introduce a queue between IoT devices and the database to buffer traffic spikes.

**Cost Consideration:**

- Queue pricing is **pay-per-request**
- ~$0.40 per **1 million requests**
- Estimated daily volume: ~300,000 requests

**Verdict:** Selected  
This approach absorbs traffic bursts without increasing fixed infrastructure costs.

---

### Critical Trade-off: Queue Type Selection

Once decoupling was chosen, the next key decision was selecting the queue type.

| Feature | Standard Queue (Chosen) | FIFO Queue |
|------|------------------------|-----------|
| Throughput | Virtually unlimited | Limited (300 / 3000 TPS) |
| Ordering | Best-effort | Strict |
| Delivery | At-least-once | Exactly-once |
| Cost | Lower | Higher |

**Decision Rationale:**

- **Throughput:** Peak traffic requires high scalability.
- **Ordering:** Log ordering is not business-critical.
- **Duplicates:** Potential duplicates can be handled at the database layer using idempotency.

Instead of paying for FIFO limitations, I chose to **shift complexity to software**, where it can be controlled more flexibly.

---

## 4. Solution Design

**Architecture Overview:**
IoT Devices -> AWS SQS (Standard) -> Python Worker -> PostgreSQL

**Key Characteristics:**

- **Buffering:**  
  The queue absorbs traffic spikes and protects the database.
- **Throttling:**  
  Workers consume messages at a steady rate the database can handle safely.
- **Scalability:**  
  Workers can be horizontally scaled during peak hours.
- **Reliability:**  
  Idempotent writes ensure data consistency.

---

## 5. Idempotency Strategy (Minimal Working Example)

To handle duplicate messages introduced by at-least-once delivery, I implemented idempotency at the database level.

### Schema Design

```sql
CREATE TABLE dispensing_logs (
    device_id VARCHAR(50),
    event_timestamp TIMESTAMP,
    drug_id VARCHAR(50),
    status VARCHAR(20),
    PRIMARY KEY (device_id, event_timestamp, drug_id)
);
-- 2. Idempotent Write Operation
INSERT INTO dispensing_logs (device_id, event_timestamp, drug_id, status)
VALUES ('DEV_001', '2026-01-25 08:05:00', 'DRUG_X', 'SUCCESS')
ON CONFLICT (device_id, event_timestamp, drug_id) 
DO NOTHING; 
-- If SQS sends this message again, the DB will silently ignore it.
-- Result: Data is consistent, even with a "messy" Queue.
'''
