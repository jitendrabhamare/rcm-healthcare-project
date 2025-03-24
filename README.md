# RCM Healthcare Data Pipeline Project

![Azure Badge](https://img.shields.io/badge/Azure-Cloud-blue) ![Data Engineering](https://img.shields.io/badge/Data-Engineering-green) ![Healthcare](https://img.shields.io/badge/Industry-Healthcare-orange)

Welcome to the **RCM Healthcare Data Pipeline Project**!

This repository contains an end-to-end data pipeline for processing Revenue Cycle Management (RCM) data from two fictional hospitals, transforming raw hospital records into actionable financial insights. Built using Microsoft Azure tools—Data Factory, Databricks, ADLS Gen2, SQL Database, Key Vault, and Unity Catalog—this project follows a **Medallion architecture** (Bronze → Silver → Gold) to automate data ingestion, cleaning, historization, and reporting.

Whether you are a Backend Software Engineer, a Data Engineer exploring Azure-based data pipelines, or a Healthcare Analyst working with RCM data, this project offers a comprehensive framework that can be leveraged for learning and adaptation.

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Achievements & Improvements](#achievements--improvements)
- [Pipeline Steps & Flow](#pipeline-steps--flow)
- [Architecture Diagram](#architecture-diagram)
- [Key Learnings](#key-learnings)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

## Overview

The pipeline consists of two main stages:

1. **Ingestion to Bronze (Raw Data Layer)**
- Extracts raw Electronic Medical Records (EMR) data from Azure SQL Databases
- Loads it into Azure Data Lake Storage Gen2 (ADLS Gen2) Bronze layer as Parquet files
- Uses metadata-driven Azure Data Factory (ADF) pipelines for automation

2. **Transformation to Silver & Gold (Processed & Business-ready Data)**
- Cleans and historizes data in Databricks (Silver Layer)
- Creates business-ready Fact and Dimension tables in Delta format (Gold Layer)

The outcome is a **scalable**, **secure**, and **fully automated data pipeline** that transforms raw, unstructured hospital data into clean, historized tables optimized for financial reporting—all documented within this repository.

![Overview Image](images/azure-de-project-stages-overview.png)

## Key Features

- **Metadata-Driven Pipelines** – A single configuration file (`load_config.csv`) orchestrates data ingestion for 10 tables, supporting:
  - **Full** loads for static data (e.g., `departments`).
  - **Incremental** loads for dynamic data (e.g., `encounters`).
- **Medallion Architecture** – Implements a layered data flow:
  - Bronze (Raw data storage in ADLS Gen2).
  - Silver (Cleansed and historized data).
  - Gold (Business-ready analytics layer).
- **Dual Transformation Logic** – Supports two data update strategies:
  - **Full Refresh** – Replaces static Silver tables (e.g., `departments`) entirely.
  - **Slowly Changing Dimension Type 2 (SCD2)** – Tracks history in dynamic tables like `patients`—preserves past records (e.g., old addresses) with `is_current` flags.
- **Delta Tables** – Silver and Gold layers utilize Delta format, enabling:
  - ACID transactions for data integrity.
  - Versioning & rollback capabilities for auditing and recovery.
- Enhanced Security with **Key Vault** – Stores sensitive credentials (e.g., SQL passwords) securely in Azure Key Vault, preventing hardcoding.
- Centralized Governance with **Unity Catalog** – Centralizes table management in Databricks—organized and governed.

<!-- ![Key Features Image](images/key_features.png) -->

## Achievements & Improvements

- **Scalability** – A single pipeline processes 10 tables using `load_config.csv`. Adding a new table requires only a config update—no code changes needed.
- **Performance Optimization** – Parallelized `pl_emr_src_to_landing` (batch count: 5), reducing ingestion time for large datasets.
- **Enhanced Flexibility** – Introduced an `is_active` flag in the config file, allowing tables to be skipped without modifying the pipeline code.
- **Security Upgrade** – Eliminated hardcoded credentials, migrating to Azure Key Vault for a production-ready, secure, and leak-proof implementation.
- **Improved Efficiency** – Mounted ADLS containers in Databricks, enabling faster access and cleaner code.

These enhancements transformed a prototype into a scalable, secure, and efficient system, ready to handle real-world RCM challenges.

## Pipeline Steps & Flow

Here’s how data moves from raw sources to polished reports:

1. **Setup Azure Resources**:
   - Resource Group: `rcm-azure-project`
   - Storage: `rcmadlsdevnew` (containers: `landing`, `bronze`, `silver`, `gold`, `configs`)
   - SQL DBs: `rcm-hospital-a`, `rcm-hospital-b` (5 tables each)
   - ADF: `rcm-health-projectdev`
   - Databricks: `rcm-hc-adb-ws`

2. **Preprocessing**:
   - Pipeline: `pl_to_insert_data_to_sql_table_preprocessing`
   - Loads CSVs (`HospitalA.csv`, `HospitalB.csv`) into SQL tables—seeds EMR data.

3. **Ingestion to Bronze**:
   - Pipelines: `pl_emr_src_to_landing` (parent), `pl_copy_from_emr` (child)
   - Reads `load_config.csv`, copies SQL to Bronze Parquet (full/incremental), logs to `audit.load_logs`.

4. **Silver Transformation**:
   - **Full Refresh**: Truncates and reloads `silver.departments`—unifies `hosa`/`hosb`, adds `is_quarantined`.
   - **SCD2**: Historizes `silver.patients`—CDM unifies schemas, two `MERGE`s track changes (e.g., address updates).

5. **Gold Transformation**:
   - Pipeline: `pl_silver_to_gold`
   - Filters Silver (`is_current = true`, `is_quarantined = false`) into Fact (`gld_transactions`) and Dimension (`gld_patients`) tables.

**Flow**:  
```
SQL DBs --> ADF (Bronze Parquet) --> Databricks (Silver Delta) --> Databricks (Gold Delta)
```

![Pipeline Flow Diagram](images/pipeline_flow.png)

## Architecture Diagram

```
+------------------+       +------------------+       +------------------+
| Azure SQL DBs    |       | ADLS Gen2        |       | Databricks       |
| (rcm-hospital-a) | ----> | Bronze (Parquet) | ----> | Silver (Delta)   |
| (rcm-hospital-b) |       |                  |       | Gold (Delta)     |
+------------------+       +------------------+       +------------------+
         |                         ^                         ^
         v                         |                         |
+------------------+       +------------------+       +------------------+
| ADF Pipelines    |       | Key Vault        |       | Unity Catalog    |
| pl_emr_src_to... | <---- | (rcm-hc-kv)     | ----> | (rcm_hc_adb_ws)  |
| pl_silver_to_gold|       +------------------+       +------------------+
```

- **SQL DBs**: Source EMR data.
- **ADF**: Moves data to Bronze, triggers Silver-to-Gold.
- **ADLS**: Stores all layers.
- **Databricks**: Transforms Bronze to Silver/Gold.
- **Key Vault**: Secures credentials.
- **Unity Catalog**: Manages tables.

![Architecture Diagram](images/architecture_diagram.png)

## Key Learnings

- **Metadata Magic**: `load_config.csv` made pipelines scalable—10 tables, one setup—automation at its best.
- **Full vs. SCD2**: Full Refresh is simple for static data; SCD2’s complexity pays off for history (e.g., patient moves).
- **Delta Power**: Delta tables brought reliability—ACID and versioning beat Parquet’s static limits.
- **Security Matters**: Key Vault turned a prototype secure—essential for production DE.
- **Troubleshooting**: Fixed errors (e.g., SQL timeouts)—DE’s half coding, half detective work.
- **Real-World Ready**: Parallel runs, mounts, and flags prepped this for scale—learned by doing!

## Project Structure

Here’s a quick guide to what’s in the repo:

- `/adf_pipelines/` - ADF pipeline JSON files (e.g., `pl_emr_src_to_landing.json`, `pl_silver_to_gold.json`).
- `/databricks_notebooks/` - Databricks notebooks (e.g., `departments_full_refresh.py`, `patients_scd2.py`).
- `/configs/` - Config files like `load_config.csv`—drives the ingestion pipeline.
- `/docs/` - Additional docs (e.g., schema definitions—if added later).
- `README.md` - You’re reading it!

This layout keeps pipelines, notebooks, and configs organized—dive into what interests you!

## Getting Started

Want to run this pipeline? Follow these steps to set it up locally or in Azure.

### Prerequisites

- **Azure Subscription**: Access to ADF, Databricks, ADLS Gen2, SQL DB, Key Vault.
- **Tools**: Azure CLI, Databricks CLI, Git.
- **Knowledge**: Basic SQL, Python, Spark—don’t worry, it’s beginner-friendly!

### Installation

1. **Clone the Repo**:
   ```bash
   git clone https://github.com/jitendrabhamare/rcm-healthcare-project.git
   cd rcm-healthcare-project
   ```

2. **Setup Azure Resources**:
   - Create resource group `rcm-azure-project`.
   - Deploy ADLS (`rcmadlsdevnew`), SQL DBs (`rcm-hospital-a`, `rcm-hospital-b`), ADF, Databricks, Key Vault via Azure Portal.

3. **Configure ADF**:
   - Import pipelines from `/adf_pipelines/` into `rcm-health-projectdev`.
   - Update linked services with your credentials (use Key Vault).

4. **Setup Databricks**:
   - Import notebooks from `/databricks_notebooks/` into `rcm-hc-adb-ws`.
   - Mount ADLS containers (e.g., `/mnt/bronze`).

5. **Upload Config**:
   - Place `load_config.csv` in `configs/emr/` in ADLS.

### Usage

1. **Run Preprocessing**:
   - Trigger `pl_to_insert_data_to_sql_table_preprocessing` with CSVs in `landing`.

2. **Ingest to Bronze**:
   - Run `pl_emr_src_to_landing`—watch Bronze populate!

3. **Transform to Silver/Gold**:
   - Execute Silver notebooks (e.g., `departments_full_refresh`, `patients_scd2`).
   - Trigger `pl_silver_to_gold` for Gold tables.

4. **Query Results**:
   - In Databricks: `SELECT * FROM silver.patients WHERE is_current = true`.

![Usage Screenshot](images/usage_screenshot.png)

## Contributing

Love to improve this? Fork the repo, make your changes, and submit a pull request. Focus areas:
- Optimize SCD2 performance.
- Add more RCM KPIs (e.g., claim denial rate).
- Enhance error logging.

## License

This project is licensed under the MIT License—see [LICENSE](LICENSE) for details.

## Contact

Questions or ideas? Reach me at:
- **GitHub**: [jitendrabhamare](https://github.com/jitendrabhamare)
- **Email**: jitendra@example.com (replace with your real email if desired)

Happy engineering!

---
