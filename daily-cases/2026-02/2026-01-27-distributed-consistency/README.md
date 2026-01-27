# 📅 Daily Case: The Case of Vanishing Orders (Distributed Consistency)

**Date:** 2026-01-27  
**Tags:** #DistributedSystems #CAPTheorem #Idempotency #DataReconciliation #FinTech

## 1. Problem Statement
During a high-traffic campaign, a **Network Partition** occurred between the `Order Service` and `Payment Service`.
- **Symptom:** Customers were charged by the bank, but no order was created in our database (Ghost Orders), or customers were charged twice due to retries.
- **Conflict:** The Finance Dashboard (Bank Data) and Sales Dashboard (Order Data) showed different revenue figures.

## 2. My Critical Thinking Process (The CAP Paradox)
During the analysis, I encountered a logical conflict regarding the **CAP Theorem**:

> **Initial Thought:** "Since financial accuracy and legal compliance are paramount, we must prioritize **Consistency (CP)**. If the network fails, the system should reject transactions to prevent double-charging or lost orders."

> **The Conflict:** However, prioritizing Consistency means closing the shop (downtime) during network blips, leading to massive revenue loss. This contradicts the business goal of maximizing sales.

> **Revised Understanding:** I realized the solution is not binary but **layered**:
> 1.  **Frontend (Live System):** Must prioritize **Availability (AP)**. Keep the shop open, accept the risk of messy data ("Eventual Consistency").
> 2.  **Backend (Data Pipeline):** Must enforce **Consistency**. My job as a Data Engineer is to clean up the mess created by the live system using **Reconciliation Pipelines**.

## 3. Engineering Decisions

### Decision 1: Defining the "Source of Truth"
**Conflict:** Order DB says "0 Sales", Bank API says "$100 Collected".
**Resolution:** I decided that **Bank/Financial Data is the Source of Truth**.
**Reasoning:** Missing an order is a bug; taking money without delivering a product is a legal liability (Fraud risk).

### Decision 2: Idempotency (The Prevention Mechanism)
**Problem:** How to prevent double-charging when a user clicks "Pay" multiple times during lag?
**Solution:** Implement **Idempotency** using a unique `Request-ID`.
- **Logic:** The client generates a unique ID (e.g., `User123-Timestamp-CartHASH`) for the button click.
- **Mechanism:** Even if the network sends the request 5 times, the backend sees the same `Request-ID` and processes the payment only **once**.



## 4. Implementation Plan (Reconciliation Pipeline)
To solve the "Eventual Consistency" gap, I designed a nightly SQL check:

```sql
-- Nightly Reconciliation Logic
-- Goal: Find payments that have no matching order (Ghost Orders)

WITH Bank_Data AS (
    SELECT payment_ref, amount, user_id FROM raw_payments_gateway
),
Internal_Orders AS (
    SELECT payment_ref, order_id FROM app_orders
)

SELECT 
    b.payment_ref,
    b.amount,
    'MISSING_ORDER' as issue_type
FROM Bank_Data b
LEFT JOIN Internal_Orders o ON b.payment_ref = o.payment_ref
WHERE o.order_id IS NULL;

-- Action: Push these results to a "Refund_Queue" or Alert the Support Team.
```

## 5. Key Learnings

### The "Two-Hat" Rule
The **Software Architect** wears the **Availability hat** — keeping the system running at all times.  
The **Data Engineer** wears the **Consistency hat** — ensuring the numbers always match and data remains correct.

---

### Idempotency
Idempotency is the mathematical guarantee that:

> f(f(x)) = f(x)

In distributed systems, this means **retrying the same request multiple times should not corrupt or change the system state**.  
This principle is critical for building reliable and fault-tolerant data pipelines.

---

### Source of Truth
In **FinTech systems**, the entity that **actually holds the money (e.g., the Bank)** is always the **source of truth**.  
Internal databases, caches, or analytics systems must be treated as **derivative**, not authoritative.
