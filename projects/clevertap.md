# Clevertap User Journey Data Pipeline

## Overview
This project automates the processing of **semi-structured user journey data** from **Clevertap**.  
We use **Clevertap APIs** to pull daily data for each source and store it in **Amazon S3** (raw zone).  
From there, **PySpark jobs on Amazon EMR** process, transform, and apply business logic before loading the results into **Amazon Redshift reporting tables**.

The entire workflow is **scheduled with Airflow**, ensuring that data is ingested, transformed, and delivered to business users in a timely manner.  
This enables near real-time decision-making and provides a single source of truth for user journey analytics.

---

## Architecture Diagram
![Clevertap Sequence Diagram](/asset/projects/clevertap/clevertap_diagram.png) 

**Flow:** Clevertap API → S3 (raw) → EMR (PySpark) → Redshift (reporting) → Dashboards  
**Control Plane:** Airflow orchestrates ingestion, transformation, and load into Redshift.

---

## Data Flow
1. **API Ingestion:** Invoke Clevertap APIs to pull daily data and land files in S3.
2. **Workflow Orchestration:** Airflow schedules jobs, tracks dependencies, and handles retries.
3. **Processing & Transformation:** PySpark on EMR processes raw data, applies business rules, and prepares curated outputs.
4. **Data Loading:** Curated data is loaded into **Redshift reporting tables** for fast querying.
5. **Consumption:** Dashboards and BI tools connect to Redshift for real-time analytics and reporting.
