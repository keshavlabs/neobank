# Neo Bank — Lakehouse Analytics Project

An end-to-end banking analytics pipeline built on Databricks using the **medallion architecture** (Bronze → Silver → Gold). Raw neobank data (customers, accounts, transactions, branches, payment gateway logs, and credit bureau reports) is ingested from a Unity Catalog volume, cleansed and deduplicated, and transformed into business-ready gold tables that power an AI/BI dashboard.

## Architecture

```
CSV files (UC Volume: /Volumes/neobank/neobank/neobank/raw)
        │
        ▼
01  Bronze  (neobank.bronze)   — raw ingest, append insert_timestamp
        │
        ▼
02  Silver  (neobank.silver)   — clean, standardize, dedupe
        │
        ▼
03  Gold    (neobank.gold)     — business aggregates & KPIs
        │
        ▼
04  AI/BI Dashboard — Neo Bank Dashboard
```

## Notebooks

| Notebook | Purpose |
| --- | --- |
| `01_Create_Banking_Tables` | Optional: recreates the reference DDL schema `neobank.banking` (branches, customers, accounts, transactions). That schema has been dropped — nothing in the pipeline depends on it; it now serves only as a data dictionary of the intended data model |
| `02_Bronze_Ingestion` | Batch ingest of raw CSVs from the UC volume into `neobank.bronze` (full overwrite) |
| `03_Silver_transform` | Cleansing (trim/casing), null filtering, and deduplication by primary key; writes `neobank.silver` |
| `04_Gold_Reports` | Builds gold tables: `branch_performance`, `customer_360`, `risk_customer_summary`, `daily_bank_kpi` |
| `05_Setting_up_metadata` | Creates the `neobank.metadata.tables` control table that drives metadata-driven ingestion |
| `06_bronze_autoloader` | Metadata-driven Auto Loader (cloudFiles) ingestion into bronze with checkpoints, `trigger(availableNow=True)` |
| `07_silver_metadata` | Metadata-driven silver transformations |
| `08_gold_run` | Runs the gold layer build |
| `Testing` | Ad-hoc testing / validation notebook |

## Dashboard

* **Neo Bank Dashboard** (AI/BI) — visualizes the gold layer: branch performance, customer segments, risk summary, and daily KPIs.

## Data Sources (raw volume)

`/Volumes/neobank/neobank/neobank/raw` with subfolders per source system:

* `branches` — 4 branches
* `customers` — 4,000 customers
* `accounts` — 4,500 accounts
* `transactions` — 15,000 transactions
* `payment_gateway_logs` — 15,000 gateway log rows
* `credit_bureau_reports` — 4,000 bureau pull records

## Unity Catalog Layout

* `neobank.bronze` — raw ingested data (as-is + `insert_timestamp`)
* `neobank.silver` — cleansed, deduplicated tables
* `neobank.gold` — business aggregates:
  * `branch_performance` — customers, accounts, deposits, and transaction totals per branch
  * `customer_360` — one row per customer with account, transaction, bureau, and segment info (High/Medium/Low Value via balance percentiles)
  * `risk_customer_summary` — customer counts, average credit score, active loans, and overdue amounts per risk grade
  * `daily_bank_kpi` — daily transaction totals cross-joined with bank-wide KPIs
* `neobank.metadata` — ingestion control table (`table_name`, `source_path`, `bronze_schema`, `source_system`, `active_flag`, `load_order`)
* `neobank.banking` — reference DDL schema from notebook 01 (**dropped**; optional to recreate via notebook 01)

## How to Run

1. Upload raw CSVs to `/Volumes/neobank/neobank/neobank/raw/<table>/`.
2. *(Optional)* Run `01_Create_Banking_Tables` to recreate the dropped `neobank.banking` reference schema.
3. Run `05_Setting_up_metadata` to register sources in the metadata control table.
4. Run `06_bronze_autoloader` → incremental bronze ingest (Auto Loader, AvailableNow trigger).
5. Run `07_silver_metadata` (or `03_Silver_transform` for the batch variant) → silver.
6. Run `08_gold_run` (or `04_Gold_Reports`) → gold tables.
7. Open the **Neo Bank Dashboard** to explore the results.

> Notebooks `02`–`04` are the original batch/overwrite version of the pipeline; `05`–`08` are the metadata-driven, incremental refresh. Either path can be used.

## Key Transformations (Silver)

* **Deduplication** — `row_number()` over primary key, keeping the most recent record (`insert_timestamp` / `txn_timestamp` / `bureau_pull_date`).
* **Standardization** — `initcap`/trim names, lower-case emails, upper-case PAN and account/txn types.
* **Null filtering** — drop rows missing the primary key.

## Customer Segmentation (Gold)

Segments are derived from percentile cutoffs on total customer balance:

* **High Value** — balance ≥ ₹469,481.11
* **Medium Value** — balance ≥ ₹269,635.01
* **Low Value** — below the medium cutoff
* **No Account** — zero balance/no accounts

## Tech Stack

* Databricks notebooks (PySpark + SQL), serverless compute
* Unity Catalog (catalog: `neobank`) with volumes, schemas, and Delta tables
* Auto Loader (`cloudFiles`) for incremental, metadata-driven ingestion
* AI/BI Lakeview dashboard for reporting

## Version Control

This project folder is backed by Git (see the `.git` directory). Commit changes from a Git folder in the Databricks workspace to push notebook and file updates.