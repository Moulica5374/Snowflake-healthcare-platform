# Snowflake Healthcare Platform

An end-to-end healthcare data pipeline built on Snowflake, dbt, Apache Airflow, and AWS S3 — using synthetic patient data modeled after real-world clinical datasets. Designed to demonstrate production-grade data engineering practices in a healthcare context.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Project Structure](#4-project-structure)
5. [Pipeline Stages](#5-pipeline-stages)
6. [Data Model](#6-data-model)
7. [Setup and Installation](#7-setup-and-installation)
8. [How to Run](#8-how-to-run)
9. [dbt Models](#9-dbt-models)
10. [Key Design Decisions](#10-key-design-decisions)
11. [Roadmap](#11-roadmap)
12. [License](#12-license)

---

## 1. Project Overview

This project simulates a real-world healthcare analytics platform that ingests, transforms, and serves clinical data for downstream reporting and analysis. It is built to mirror the kind of data engineering work done in payer and provider organizations — member eligibility, clinical encounters, diagnoses, and medication history — using a modern cloud data stack.

**Goals:**
- Demonstrate end-to-end pipeline engineering from raw data ingestion to analytics-ready Gold tables
- Apply Medallion architecture (Bronze → Silver → Gold) within Snowflake
- Use dbt for transformation with full lineage, tests, and documentation
- Orchestrate the full pipeline with Apache Airflow

---

## 2. Architecture

```
Synthea (Synthetic Patients)
        │
        ▼
   AWS S3 (Raw Landing Zone)
        │
        ▼  [Snowpipe — continuous ingestion]
   Snowflake — BRONZE (raw, unmodified CSVs)
        │
        ▼  [dbt staging models]
   Snowflake — SILVER (cleaned, typed, deduplicated)
        │
        ▼  [dbt mart models]
   Snowflake — GOLD (analytics-ready fact & dimension tables)
        │
        ▼
   BI / Reporting Layer (Tableau / PowerBI / Snowsight)
        
   Apache Airflow — orchestrates all stages end to end
```

---

## 3. Tech Stack

| Layer | Tool |
|---|---|
| Data generation | [Synthea](https://github.com/synthetichealth/synthea) |
| Raw storage | AWS S3 |
| Data warehouse | Snowflake |
| Ingestion | Snowpipe (S3 → Snowflake) |
| Transformation | dbt Core |
| Orchestration | Apache Airflow |
| Language | Python 3.10+ |
| IaC (planned) | Terraform |
| CI/CD (planned) | GitHub Actions |

---

## 4. Project Structure

```
snowflake-healthcare-platform/
├── dags/                        # Airflow DAGs
│   └── healthcare_pipeline.py
├── dbt/                         # dbt project
│   ├── models/
│   │   ├── staging/             # Bronze → Silver
│   │   ├── intermediate/        # Silver joins and enrichment
│   │   └── marts/               # Gold — fact & dim tables
│   ├── tests/
│   ├── macros/
│   └── dbt_project.yml
├── ingestion/                   # Snowpipe setup + S3 loader scripts
│   ├── snowpipe_setup.sql
│   └── upload_to_s3.py
├── sql/                         # Utility and setup SQL
│   ├── create_schemas.sql
│   └── warehouse_config.sql
├── synthea/                     # Synthea config + generated data (gitignored)
│   └── synthea_config.properties
├── docs/                        # Architecture diagrams, design notes
├── .env.example
├── requirements.txt
└── README.md
```

---

## 5. Pipeline Stages

**Bronze — Raw ingestion**
Raw CSV files from Synthea land in S3 and are loaded into Snowflake via Snowpipe with no transformation. Schema-on-read, all columns as `VARCHAR`.

**Silver — Cleaned and typed**
dbt staging models cast columns to proper types, deduplicate records, apply null handling, and standardize field names following snake_case conventions.

**Gold — Analytics-ready**
dbt mart models build fact tables (encounters, claims) and dimension tables (patients, providers, conditions) following a star schema pattern suitable for BI consumption.

---

## 6. Data Model

Core Synthea tables used in this pipeline:

| Table | Description |
|---|---|
| `patients` | Demographics — DOB, gender, race, zip code |
| `encounters` | Clinical visits — type, date, provider, cost |
| `conditions` | Diagnoses — SNOMED codes, onset/stop dates |
| `medications` | Prescriptions — RxNorm codes, dosage, cost |
| `observations` | Lab results and vitals — LOINC codes |
| `procedures` | Procedures performed — SNOMED, cost |
| `providers` | Provider details — specialty, organization |
| `organizations` | Facility details — name, location |

---

## 7. Setup and Installation

### Prerequisites

- Python 3.10+
- Snowflake account (free trial works)
- AWS account with S3 access
- Apache Airflow 2.x
- dbt Core (`pip install dbt-snowflake`)
- Java 11+ (for Synthea)

### Environment variables

Copy `.env.example` to `.env` and fill in your credentials:

```bash
SNOWFLAKE_ACCOUNT=
SNOWFLAKE_USER=
SNOWFLAKE_PASSWORD=
SNOWFLAKE_ROLE=
SNOWFLAKE_WAREHOUSE=
SNOWFLAKE_DATABASE=
SNOWFLAKE_SCHEMA=

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_BUCKET_NAME=
```

### dbt profile

Add to `~/.dbt/profiles.yml`:

```yaml
snowflake_healthcare:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      database: HEALTHCARE_DB
      schema: BRONZE
      threads: 4
```

---

## 8. How to Run

### Step 1 — Generate synthetic patient data

```bash
cd synthea
./run_synthea -p 1000   # generates 1000 patients
```

### Step 2 — Upload to S3

```bash
python ingestion/upload_to_s3.py --source synthea/output/csv --bucket $S3_BUCKET_NAME
```

### Step 3 — Set up Snowflake schemas and Snowpipe

```bash
snowsql -f sql/create_schemas.sql
snowsql -f ingestion/snowpipe_setup.sql
```

### Step 4 — Run dbt transformations

```bash
cd dbt
dbt deps
dbt run
dbt test
dbt docs generate && dbt docs serve
```

### Step 5 — Trigger Airflow DAG

```bash
airflow dags trigger healthcare_pipeline
```

---

## 9. dbt Models

```
staging/
  stg_patients.sql
  stg_encounters.sql
  stg_conditions.sql
  stg_medications.sql
  stg_observations.sql

intermediate/
  int_patient_encounters.sql
  int_condition_history.sql

marts/
  dim_patients.sql
  dim_providers.sql
  fact_encounters.sql
  fact_medications.sql
```

All models include:
- Column-level descriptions in `schema.yml`
- `not_null` and `unique` tests on primary keys
- `accepted_values` tests on categorical fields

---

## 10. Key Design Decisions

**Why Snowpipe?** Enables event-driven, near-real-time ingestion from S3 without polling. Scales automatically and decouples ingestion from transformation.

**Why dbt?** Transforms SQL into version-controlled, testable, documented assets. The lineage graph makes the Bronze → Silver → Gold flow explicit and auditable.

**Why Medallion architecture?** Separates concerns cleanly — raw data is never overwritten, Silver is the trusted cleaned layer, Gold is purpose-built for consumption.

**SCD Type 2 (planned):** Patient and provider dimensions will use dbt snapshots to track slowly changing attributes over time — mirroring real payer data practices.

**Cost control:** All dev workloads use an XS Snowflake warehouse with auto-suspend after 60 seconds of inactivity.

---

## 11. Roadmap

- [ ] Snowpipe → Kafka connector for real-time streaming ingestion
- [ ] Snowflake ML — readmission risk scoring model
- [ ] dbt exposures for BI layer documentation
- [ ] GitHub Actions CI/CD for dbt test + deploy on PR merge
- [ ] Terraform for Snowflake warehouse and role provisioning
- [ ] FHIR R4 schema support

---

## 12. License

MIT License. See [LICENSE](LICENSE) for details.

---

*Built by [Moulica](https://github.com/Moulica5374) as a portfolio project demonstrating production-grade healthcare data engineering on a modern cloud data stack.*