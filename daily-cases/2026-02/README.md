# 📅 Daily Case: The Silent Poisoning (Data Quality & DLQ)

**Date:** 2026-01-29  
**Tags:** #DataQuality #DeadLetterQueue #DataContracts #Reliability #AdTech

## 1. Problem Statement
An upstream bug in the Frontend caused `click_id` fields to arrive as `NULL` or `undefined` in the real-time clickstream.
- **Risk:** Strict database constraints (`NOT NULL`) could cause the entire batch processing pipeline to fail, blocking valid revenue-generating data.
- **Impact:** Inaccurate billing and potential revenue loss if the pipeline stops or if bad data pollutes the analytics tables.

## 2. Engineering Analysis & Decisions

### Decision 1: Validation Strategy (Fail-Fast vs. Quarantine)
**Scenario:** A batch of 10,000 records contains 1 bad record.
**Decision:** Avoid "Fail-Fast" at the pipeline level. Adopt the **Dead Letter Queue (DLQ)** pattern.
**Reasoning:**
- Failing the whole batch for one error halts the business (High Availability risk).
- We must process the 9,999 valid records immediately while isolating the error.

### Decision 2: The Dead Letter Queue (DLQ) Architecture
**Mechanism:**
1.  **Validate:** Check schema rules (e.g., `click_id IS NOT NULL`) in memory during transformation.
2.  **Route:**
    - **Valid Data:** Write to `Silver_Clicks` table.
    - **Invalid Data:** Write to `DLQ_Clicks` table with metadata (`error_reason`, `timestamp`, `original_payload`).

### Decision 3: Remediation Strategy (Reprocessing)
**Question:** What to do with the bad data in DLQ?
**Decision:** **Reprocess/Replay**.
**Reasoning:**
- In AdTech/Finance, data is money. We cannot simply "ignore" or delete invalid rows.
- **Workflow:** Once the root cause is identified (e.g., ID location changed in JSON), write a recovery script to fix the data in DLQ and push it back into the `Silver` layer.

### Decision 4: Prevention with Data Contracts (Schema Registry)
**Problem:** How to prevent the upstream team (Frontend) from changing data structure silently?
**Decision:** Implement **Schema Registry** with strict formats like **Avro** or **Protobuf**.
**Reasoning:**
- Unlike JSON, formats like Avro enforce a strict schema.
- If Frontend tries to send a payload without `click_id`, the Schema Registry rejects the event at the source (API Gateway level).
- The bad data never even enters our Data Pipeline, saving compute resources.

## 3. Implementation Logic (Pseudo-Python)

```python
def process_batch(batch_df):
    valid_rows = []
    invalid_rows = []
    
    for row in batch_df:
        # Validation Logic
        if row['click_id'] is not None and row['click_id'] != "undefined":
            valid_rows.append(row)
        else:
            # Enrich with error metadata for debugging
            row['error_reason'] = 'MISSING_CLICK_ID'
            row['ingest_time'] = now()
            invalid_rows.append(row)
    
    # 1. Commit valid data to Main Table (Business continues)
    write_to_silver(valid_rows)
    
    # 2. Quarantine invalid data (For later fix)
    write_to_dlq(invalid_rows)
    
    # 3. Alert if DLQ grows too fast
    if len(invalid_rows) > threshold:
        send_slack_alert("High Data Quality Failure Rate in DLQ!")
```

## 4. Key Learnings
- **Don't Stop the Line:** In production pipelines, reliability comes from isolating errors, not crashing on them.
- **Data is Asset:** Never drop bad data silently. Quarantine it in a DLQ for inspection and recovery.
- **Contracts Matter:** Use Schema Registries (Avro/Protobuf) to act as a "police" at the API gate, preventing bad data from entering the system in the first place.
