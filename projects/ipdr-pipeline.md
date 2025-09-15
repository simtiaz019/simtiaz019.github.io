# IPDR Data Processing Pipeline

## What is an IPDR File?
**IPDR (Internet Protocol Detail Record)** files are detailed logs that capture subscribers’ data session activity on a telecom network. Each record contains session start/end times, data volume transferred, subscriber identifiers (e.g., MSISDN), and other key attributes.  
These files are critical for:
- **Billing & Revenue Assurance** – ensuring accurate charging for usage
- **Regulatory Compliance** – generating lawful interception and reporting
- **Customer Behavior Analysis** – understanding data consumption patterns
- **Network Optimization** – detecting congestion and planning capacity

---

## Architecture Overview

![IPDR Pipeline Diagram](/asset/projects/ipdr-pipeline/ipdr_diagram.png)

The above diagram represents our **IPDR data pipeline**, which follows the **Medallion Architecture**. This design uses three progressive layers (**L1 → L2 → L3**) to ensure data quality, consistency, and scalability.

---

### 1. Data Ingestion – Landing Zone
- IPDR files are received from the **IN (Intelligent Network)** into the **DMP (Data Management Platform) Landing Zone**.
- This zone acts as the staging area before processing begins.

### 2. L1 – Raw Data Layer
- **L1 Spark Job** runs continuously to:
  - Register files into the metadata catalog.
  - Load the unmodified raw data into **L1 Hive tables**.
- This layer ensures a **single source of truth** and maintains historical copies of all received files.

### 3. L2 – Transformation Layer
- **L2 Spark Job** runs **every 20 minutes**.
- It processes raw data from L1, applies parsing, enrichment, and business rules, and writes the cleaned output into **L2 Hive tables**.
- This ensures near real-time availability of processed data for reporting and downstream applications.

### 4. L3 – Reporting Layer
- **L3 Spark Job** runs **once per day**.
- It aggregates L2 data, generates final **reporting tables**, and publishes them to:
  - **BI Tools** for analytics
  - **Teradata** for storage and querying
  - **Dashboards** for business visualization and decision-making

---

## Role of Airflow and Spark
- **Apache Airflow** orchestrates the entire workflow, handling:
  - Task scheduling (e.g., triggering L1 jobs immediately, L2 every 20 minutes, L3 daily)
  - Dependencies between stages (L2 waits for L1, L3 waits for L2)
  - Failure handling and notifications (alerts for SLA breaches)

- **Apache Spark** provides the distributed processing engine:
  - Enables parallel processing of 200K+ daily files efficiently.
  - Scales horizontally across the cluster for high-performance data transformation.
  - Supports complex ETL logic at scale.

---

## Medallion Architecture Benefits
- **L1 (Bronze Layer):** Retains raw, immutable data for replay/debugging.
- **L2 (Silver Layer):** Cleansed and structured data, ready for analytics.
- **L3 (Gold Layer):** Business-level aggregated data, optimized for consumption.

This layered approach improves **data reliability, reusability, and auditability**.

---

## Challenges & Solutions

### Challenge: Small File Problem
- With **200K+ IPDR files ingested daily**, many files are very small (few KBs).
- Spark suffers from performance degradation because:
  - Each small file becomes a separate task.
  - High overhead is spent on task scheduling and metadata operations instead of actual computation.
  - Leads to slow job execution and excessive memory usage.

### Solution:
- **File Compaction:** Merge multiple small files into larger Parquet/ORC files in L1 using a pre-processing job.
- **Partitioning Strategy:** Use date/hour-based partitioning in Hive tables to avoid unnecessary scans.
- **Auto Loader / Structured Streaming:** Incrementally load and compact data as it arrives.
- **Adaptive Parallelism:** Tune `spark.sql.shuffle.partitions` and enable dynamic allocation for efficient cluster resource utilization.
