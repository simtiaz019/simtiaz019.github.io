# DMP Trigger Framework

## Overview
The **DMP Trigger Framework** was developed to modernize and scale the legacy DMP trigger system.  
Initially, the system was capable of processing around **1 million events per day**, but with exponential growth in data and event sources, we needed a robust, scalable solution.  

Today, the new framework is capable of processing **1+ billion events per day** across multiple data sources.  
It supports **31 triggers (as independent microservices)** running on top of **Kafka** and **Apache Ignite** clusters, ensuring scalability, resilience, and operational efficiency.  

---

## Architecture

![DMP Trigger Framework](/asset/projects/dmp-triggers/dmp_trigger.png) 

The framework is designed around three main components: **Source**, **Trigger Framework**, and **Destination**.

### 1. Source
- **IN, ADM, CDRs, SMPP, App** act as data producers.
- Two types of input data:
  1. **Real-Time (WebHook based)** – Events ingested instantly from upstream systems.
  2. **Near Real-Time (Batch/ CDR based)** – Events ingested periodically in batches.

Each data type is pushed to dedicated **Kafka topics** for parallel and isolated processing.

### 2. Trigger Framework
This is the core of the architecture, consisting of multiple layers and technologies:

- **Kafka**  
  Acts as the backbone of the framework.  
  - Collects and distributes all events from multiple sources.  
  - Maintains separate topics for each event type.

- **Web Hook & Batch Ingestion**  
  Responsible for ingesting both real-time and near real-time data into Kafka topics.

- **Trigger Business Logic**  
  - 31 independent triggers (microservices) run on top of **Spring Boot**.
  - Consume data from Kafka topics, process according to defined rules.
  - Utilize **Apache Ignite** for in-memory computation and distributed caching.

- **Ignite**  
  - Stores intermediate state of triggers and enables fast lookups.
  - Helps perform aggregations and deduplication before generating final trigger output.

- **Output Stage**  
  Processed events are published back to dedicated Kafka topics for consumption by downstream systems.

- **Monitoring & Logging**  
  Integrated with **Prometheus** and **Grafana** for real-time metrics, alerting, and SLA compliance.
  Logging and error handling are centralized to reduce manual operational overhead.

### 3. Destination
Final processed events are consumed by various business applications:
- **App** (Mobile or Web)
- **CMS** (Content Management System)
- **LMS** (Loyalty Management System)
- **MFS** (Mobile Financial Services)
- **Digital Platforms** (Campaigns, Analytics)

This allows downstream systems to respond in real-time (e.g., sending notifications, triggering offers, enabling workflows).

---

## Key Benefits
- **High Scalability:** Supports billions of events daily with minimal latency.
- **Modular Design:** Easy to add new triggers without changing core framework.
- **Real-Time & Near Real-Time Processing:** Handles both webhook-driven and batch-driven data efficiently.
- **Reduced Manual Work:** Automated monitoring, logging, and alerting reduce operational effort.
- **Developer Friendly:** Provides a base framework for quick development of new triggers.

---

## Challenges & Lessons Learned
- **Scaling Event Volume:** Moving from 1M to 1B+ daily events required redesigning Kafka partitioning strategy and Ignite cluster sizing.
- **Small Event Bursts:** Webhook-based events could cause sudden spikes; solved using Kafka buffering and back-pressure handling.
- **Monitoring Complexity:** With 31 microservices running simultaneously, observability became critical — addressed via Prometheus + Grafana dashboards and alerting rules.

