# Use Apache Iceberg for Bronze / Silver / Gold Medallion Pattern for Data Lake architecture  

## Context

We are designing a data lake architecture on cloud object storage (e.g., GCS, S3, ADLS) to support both streaming ingestion and analytical queries. We want to follow the Medallion architecture (Bronze/Silver/Gold) for better data quality management:
- Bronze (Raw): Ingested data, minimally processed, often as-is from Kafka, Pub/Sub, or CDC sources.
- Silver (Cleansed/Refined): Data standardized, deduplicated, schema-aligned, enriched, but still relatively close to source.
- Gold (Curated/Business): Aggregated, business-ready, KPI tables optimized for BI/ML workloads.

Challenges with a naive approach (plain Parquet files per layer):
- No transactional guarantees (risk of partial reads during writes).
- Hard to evolve schemas when sources change.
- Partition management and metadata handling become brittle.
- Difficult deletes (e.g., GDPR) and upserts (CDC, deduplication).
- Inconsistent data between layers if multiple writers/readers overlap.

We need a table format that:
- Provides ACID transactions across layers.
- Supports time travel for debugging and reproducibility.
- Allows schema/partition evolution over time.
- Enables multi-engine access (Spark, Flink, Trino, DuckDB, PyIceberg).

## Decision

We will implement the Medallion architecture using Apache Iceberg tables:
- Bronze layer → Iceberg “raw” tables
    - Directly ingest events or CDC changes into append-only Iceberg tables.
    - Store data in Parquet/Avro under the Iceberg metadata layer.
    - Benefit: safe streaming writes (Flink/Spark structured streaming) with atomic commits.
- Silver layer → Iceberg “refined” tables
    - Transform Bronze data into cleansed, deduplicated, schema-aligned Iceberg tables.
    - Use Iceberg MERGE INTO to handle late arrivals, updates, deletes.
    - Partition evolution (e.g., from daily → hourly) can be applied without rewriting all data.
- Gold layer → Iceberg “curated” tables
    - Aggregate/refine Silver into business-level KPIs, fact/dimension tables, or ML feature stores.
    - Benefit: time travel allows reproducing reports “as of” a past snapshot.

All layers will be registered in a central Iceberg catalog (e.g., REST, Glue, Nessie, SQL catalog) to enable consistent discovery and access from Trino/Spark/Flink.

Schedule compaction/optimization jobs to merge small files in Bronze and Silver.

Use snapshot expiration to manage storage cost while retaining reproducibility windows.

## Consequences

Positive
- Provides ACID guarantees across all medallion layers.
- Time travel supports debugging: Silver can be rebuilt from Bronze snapshots, Gold from Silver.
- Supports multi-engine queries without duplicating data.
- Simplifies schema and partition evolution compared to plain Parquet.
- Enables row-level deletes and updates for regulatory compliance.

Negative
- Requires operational overhead (catalog service, compaction/vacuum jobs).
- More complex learning curve for developers vs. plain files.
- Storage usage may increase due to multiple snapshots and lineage.
- Write performance can be lower than raw file dumps (because of transaction/manifest overhead).

## Alternatives Considered

- Plain Parquet/ORC files: simpler, but no ACID, no schema evolution, unsafe concurrent writes.
- Delta Lake: similar features, but ecosystem less engine-agnostic.
- Hudi: stronger CDC/upsert handling, but weaker in multi-engine interoperability.