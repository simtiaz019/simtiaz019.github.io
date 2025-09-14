# IPDR Data Pipeline — Architecture & Implementation

This pipeline ingests raw IPDR data, normalizes it, and publishes curated data to downstream consumers.

![System Diagram](asset/projects/ipdr-pipeline/diagram.png)

## Highlights
- 6TB+ daily (40B+ records)
- Airflow for orchestration, Delta for ACID+time travel

```python
# Sample Spark step
df = (spark.read.format("parquet").load(input_path)
      .dropDuplicates(["session_id"])
      .repartition(200))
df.write.mode("append").format("delta").save(output_path)


## Highlights 
this is the conclution. 