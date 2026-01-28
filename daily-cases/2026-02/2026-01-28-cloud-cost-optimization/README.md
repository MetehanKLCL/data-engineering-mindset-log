# 📅 Daily Case: Cloud Cost Optimization (The Format Wars)

**Date:** 2026-01-28  
**Tags:** #AWS #BigData #Parquet #Athena #CostOptimization #DataLake

## 1. Problem Statement
Our Data Lake stores raw logs in **JSON** format on S3.
- **Symptom:** AWS Athena costs are skyrocketing because every query scans the entire dataset (Full Table Scan).
- **Goal:** Reduce S3 storage costs and Athena query costs by at least 70% without losing data fidelity.

## 2. Engineering Analysis & Decisions

### Decision 1: File Format (Row vs. Columnar)
**Legacy:** JSON (Row-based). Good for human readability, terrible for analytics.
**Decision:** Convert to **Apache Parquet**.
**Reasoning:**
- **Columnar Storage:** Analytics queries usually ask for specific columns (e.g., `SELECT count(likes)...`). Parquet allows reading *only* the required column, skipping the rest.
- **Compression:** Parquet (Snappy/GZIP) typically reduces file size by 6x-10x compared to JSON.

### Decision 2: Partitioning Strategy (Directory Structure)
**Legacy:** Flat structure or simple dates (`/2026-01-28/`).
**Decision:** Implement **Hive-Style Partitioning** (`/year=2026/month=01/day=28/`).
**Reasoning:**
- Enables **Partition Pruning**.
- When filtering by date, the query engine skips scanning 99% of the folders.
- Virtual Columns: The directory path itself becomes data, saving storage inside the file.

### Decision 3: ROI Analysis (The Trade-off)
**Question:** Is the compute cost of running an ETL job (to convert JSON -> Parquet) worth it?
**Math:**
- **JSON:** Zero ETL cost, but **High Read Cost** (Scanning 1TB costs $5).
- **Parquet:** Low ETL cost (e.g., $2 compute), but **Ultra-Low Read Cost** (Scanning compressed columns costs $0.05).
- **Result:** Since data is written once but queried hundreds of times ("Write Once, Read Many"), the ETL investment pays off almost immediately.

## 3. Implementation Plan (AWS Glue / PySpark)

```python
# Conceptual PySpark ETL Job
from pyspark.sql.functions import year, month, dayofmonth

# 1. Read Raw JSON (Expensive Step)
df = spark.read.json("s3://raw-bucket/logs/")

# 2. Transform & Add Partition Columns
df_transformed = df.withColumn("year", year("timestamp")) \
                   .withColumn("month", month("timestamp")) \
                   .withColumn("day", dayofmonth("timestamp"))

# 3. Write as Parquet with Hive Partitioning (Optimization Step)
df_transformed.write \
    .mode("append") \
    .partitionBy("year", "month", "day") \
    .parquet("s3://silver-bucket/optimized_logs/")
```

## 4. Key Learnings

* **Columnar > Row:** For analytical workloads (OLAP), never use JSON/CSV. Always use **Parquet** or **ORC**.
* **Partitioning is Key:** A good folder structure is as important as the code itself. Proper partitioning is the first line of defense against high query costs.
* **ROI (Return on Investment):** Optimization is an investment, not a cost. Spending CPU cycles to compress and structure data saves massive I/O costs and time in the long run.
