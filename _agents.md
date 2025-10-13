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

# Adopt Apache Iceberg REST Catalog for all Iceberg table deployments

## Context

Usual Iceberg table deployments use a mix of catalog mechanisms (HadoopCatalog, Hive Metastore, Glue). While these solutions work, they exhibit several limitations:
- Tight coupling to specific compute engines (e.g., Spark ↔ Hive).
- Limited interoperability between engines (Spark, Flink, Trino, Dremio).
- Difficult authentication and authorization management, especially in multi-cloud or multi-tenant setups.
- Operational friction — separate catalog backends per project, inconsistent namespace handling.
- Vendor-specific APIs, leading to lock-in or re-implementation effort when migrating.
- It is also difficult to query such catalogs using standalone clients.

The Iceberg REST Catalog specification (part of Apache Iceberg 1.3+) introduces a standardized, HTTP-based API for all catalog operations (create, list, load, commit). This model decouples client configuration from metadata storage and enables a clean separation of concerns.

## Decision

- We will adopt the Iceberg REST Catalog as the default catalog interface for all new and migrated Iceberg tables.
- All compute engines (Spark, Flink, Trino, Dremio, etc.) will connect via the REST Catalog API.
- The REST endpoint will abstract away the backend metadata store (PostgreSQL, Glue, etc.).

## Consequences

✅ Positive
- Full multi-engine interoperability: any Iceberg-compliant client can access tables through REST.
- Centralized governance: unified namespace and auth model across teams.
- Cloud-agnostic: same API works on GCP, AWS, or on-prem.
- Future-proof: aligns with Iceberg’s official specification direction.
- Simplified client config: engines point to a single HTTP endpoint instead of DB or Hive URIs.
- Decoupled lifecycle: catalog server can evolve independently from storage or compute.

⚠️ Negative
- Requires operating a catalog service (extra deployment component).
- Latency overhead (minor HTTP call vs local Hive Metastore).
- Migration effort from legacy catalogs.
- Security and SLA responsibility moves to us (until managed offerings mature).

## Options Considered
- Hadoop Catalog: Filesystem-based metadata. Simple, no service. No multi-engine consistency, limited auth.
- Hive Metastore: HMS catalog. Familiar, mature. Tight coupling, Kerberos, legacy stack.
- Glue Catalog: AWS managed	Easy for AWS. AWS-only, limited interop.
- REST Catalog (chosen): Iceberg REST spec. Open, extensible, secure. New operational component.