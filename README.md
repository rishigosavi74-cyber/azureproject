# 🎧 Spotify End-to-End Azure Data Engineering Pipeline

An end-to-end data engineering project that simulates a Spotify-style streaming analytics platform, built on **Azure Data Factory**, **Azure Data Lake Storage Gen2**, and **Azure Databricks**, following the **Medallion Architecture** (Bronze → Silver → Gold).

The pipeline ingests raw source data incrementally, processes it through structured streaming transformations, and produces business-ready, query-optimized tables governed by **Unity Catalog**.

---

## 📌 Project Overview

This project demonstrates a production-style data pipeline that:

- Ingests data incrementally from a SQL source into cloud storage
- Cleans, validates, and transforms raw data using distributed processing
- Builds a star-schema-style Gold layer ready for analytics/BI
- Applies data governance and access control
- Uses dynamic SQL generation to keep transformation logic scalable and maintainable

---

## 🏗️ Architecture

```
 SQL Source (MySQL/SFTP)
        │
        ▼
 Azure Data Factory (Incremental Load — Watermark Strategy)
        │
        ▼
 ┌─────────────────────────────────────────────┐
 │           Azure Data Lake Storage Gen2        │
 │                                               │
 │   Bronze (Raw)  →  Silver (Cleaned)  →  Gold  │
 │                                               │
 └─────────────────────────────────────────────┘
        │                    │                │
   Autoloader          PySpark          Databricks ETL Pipeline
   (cloudFiles)      Transformations    (Delta Live Tables +
        │                    │           Jinja2-templated
        │                    │           Dynamic SQL Joins)
        ▼                    ▼                ▼
              Azure Databricks + Unity Catalog
              (Governance, Access Control, Delta Lake)
```

---

## 🔧 Tech Stack

| Layer | Technology |
|---|---|
| Orchestration & Ingestion | Azure Data Factory (ForEach, Lookup, Incremental Load) |
| Storage | Azure Data Lake Storage Gen2 (Bronze / Silver / Gold containers) |
| Processing | Azure Databricks, PySpark Structured Streaming, Autoloader |
| Table Format | Delta Lake |
| Governance | Unity Catalog |
| Dynamic SQL Generation | Jinja2 templating |
| Version Control / CI-CD | GitHub, Databricks Asset Bundles (DAB) |

---

## 🥉🥈🥇 Medallion Architecture Breakdown

### Bronze Layer — Raw Ingestion
- Azure Data Factory pulls data incrementally from the SQL source using a **watermark-based strategy** (only new/changed records are pulled each run, not the full dataset).
- Raw files land in ADLS Gen2 with no transformation, preserving full fidelity of the source.

### Silver Layer — Cleaned & Validated
- **Databricks Autoloader** (`cloudFiles`) incrementally and automatically detects and streams new files from Bronze.
- Schema evolution handled via `schemaEvolutionMode`.
- Transformations applied:
  - Deduplication (`dropDuplicates`)
  - Column standardization (casing, dropping rescued/metadata columns)
  - Derived fields (e.g. duration buckets, flags)
- Data written as managed Delta tables per dimension:
  - `DimUser`, `DimArtist`, `DimTrack`, `DimDate`, `FactStream`

### Gold Layer — Business-Ready Star Schema
- Built using a **Databricks ETL pipeline** that reads the curated Silver dimension and fact tables and prepares them for consumption.
- Implemented using **Delta Live Tables (DLT)** for declarative, dependency-aware orchestration of the Gold layer transformations (`@dlt.table` definitions per staging/output table), giving automatic lineage tracking and pipeline health monitoring out of the box.
- **Jinja2 templating** is used to dynamically generate the multi-table JOIN SQL across all dimension tables from a configurable parameter list — new dimensions can be added to the star schema by updating a config list, without hand-rewriting SQL each time.
- Output tables are optimized, denormalized, and ready for downstream BI/reporting use cases.

---

## 🔐 Governance — Unity Catalog

- All Bronze/Silver/Gold data is registered under a single **Unity Catalog metastore**, giving a unified 3-level namespace (`catalog.schema.table`) across the project.
- Access to underlying storage is managed through **Azure Managed Identity–based storage credentials** and **external locations**, rather than exposing raw storage keys.
- Enables fine-grained permissioning and a single source of truth for data discovery.

---

## ⚙️ CI/CD

- Databricks notebooks and pipeline logic are version-controlled via **GitHub**.
- **Databricks Asset Bundles (DAB)** used to package and deploy pipeline artifacts consistently across environments.

---

## 💡 Engineering Challenges Solved

- **Incremental ingestion design** — implemented watermark-based delta loads in ADF to avoid full reloads on every run.
- **Streaming checkpoint management** — designed isolated checkpoint locations for previewing streaming data vs. production writes, avoiding checkpoint collisions.
- **Unity Catalog infrastructure recovery** — diagnosed and resolved a corrupted metastore root storage credential (`DAC_DOES_NOT_EXIST`) by provisioning a new metastore, re-registering external locations, and migrating all Silver tables — restoring the pipeline without any data loss.
- **DLT Serverless compatibility issue** — the Gold layer DLT pipeline failed under Serverless compute due to Unity Catalog credential resolution; resolved by reconfiguring the pipeline to run on a classic autoscaling cluster, after which the pipeline executed end-to-end successfully.
- **Dynamic query generation** — replaced hardcoded multi-table JOIN SQL with a Jinja2-templated approach driven by a parameter list, making the Gold layer easily extensible.

---

## 📂 Repository Structure

```
├── adf-pipelines/           # Azure Data Factory pipeline JSON definitions
│   ├── incremental_ingestion/
│   └── incremental_loop/
├── databricks-notebooks/    # PySpark notebooks (Bronze → Silver → Gold)
│   ├── silver/
│   └── gold/
├── jinja-templates/         # Dynamic SQL generation for Gold layer joins
├── docs/                    # Architecture diagrams and documentation
└── README.md
```

---

## 🚀 Future Improvements

- Add automated data quality checks (e.g. Great Expectations / DLT expectations)
- Add orchestration monitoring & alerting (ADF alerts / Databricks Jobs notifications)
- Extend Gold layer with additional business marts

---

## 👤 Author

**Rushi Gosavi**
Aspiring Data Engineer | Azure • Databricks • PySpark
