# SQL Data Warehouse Project

A modern data warehouse built on SQL Server, following the **Medallion Architecture** (Bronze → Silver → Gold). It ingests raw CRM and ERP data from CSV files, cleans and standardizes it, and models it into a star schema ready for analytics and reporting.

## Architecture

```
Source Files (CSV)          Bronze              Silver               Gold
┌─────────────┐        ┌──────────────┐   ┌──────────────────┐  ┌──────────────────┐
│ source_crm/ │  ────▶ │ Raw, as-is   │──▶│ Cleaned,         │─▶│ Business-ready   │
│ source_erp/ │        │ tables       │   │ standardized,    │  │ star schema      │
│             │        │ (BULK INSERT)│   │ deduplicated     │  │ (views)          │
└─────────────┘        └──────────────┘   └──────────────────┘  └──────────────────┘
```

- **Bronze** — raw data loaded as-is from source CSVs, no transformations
- **Silver** — cleaned, standardized, deduplicated, and business-rule-corrected data
- **Gold** — final dimension and fact views, modeled as a star schema, ready for BI tools and ad-hoc analysis

### Source systems

| Source | Tables |
|---|---|
| CRM | `cust_info`, `prd_info`, `sales_details` |
| ERP | `loc_a101`, `cust_az12`, `px_cat_g1v2` |

### Gold layer (star schema)

- `gold.dim_customers` — customer dimension (merges CRM + ERP customer/location data)
- `gold.dim_products` — product dimension (merges CRM product info with ERP category data, current products only)
- `gold.fact_sales` — sales fact table, joined to both dimensions via surrogate keys

## Tech stack

- Microsoft SQL Server (T-SQL)
- `BULK INSERT` for CSV ingestion
- Stored procedures for repeatable ETL loads
- Views for the Gold semantic layer

## Project structure

```
datasets/               # Source CSV files (source_crm/, source_erp/)
scripts/
├── init_database.sql    # Creates the DataWarehouse database + bronze/silver/gold schemas
├── bronze/
│   ├── ddl_bronze.sql        # Bronze table definitions
│   └── proc_load_bronze.sql  # Loads raw CSVs into Bronze via BULK INSERT
├── silver/
│   ├── ddl_silver.sql        # Silver table definitions
│   └── proc_load_silver.sql  # Cleans/transforms Bronze data into Silver
└── gold/
    └── ddl_gold.sql           # Gold star-schema views (dims + fact)
tests/                  # Data quality / validation checks
docs/                   # Project documentation
```

## Getting started

### Prerequisites

- SQL Server (2019+ recommended) or SQL Server running in a container
- SQL Server Management Studio (SSMS) or another SQL client
- The source CSV files placed under `datasets/source_crm/` and `datasets/source_erp/`

### Setup

1. **Create the database and schemas**

   Run [`scripts/init_database.sql`](scripts/init_database.sql). This drops and recreates a `DataWarehouse` database and creates the `bronze`, `silver`, and `gold` schemas.

   >  This script drops the `DataWarehouse` database if it already exists. Back up any existing data first.

2. **Create Bronze tables and load raw data**

   ```sql
   -- Create tables
   :r scripts/bronze/ddl_bronze.sql

   -- Create and run the load procedure
   :r scripts/bronze/proc_load_bronze.sql
   EXEC bronze.load_bronze;
   ```

   `proc_load_bronze.sql` uses `BULK INSERT` with hardcoded local file paths — **update the file paths in the script to match where you've placed the CSVs on your machine** before running it.

3. **Create Silver tables and transform the data**

   ```sql
   :r scripts/silver/ddl_silver.sql
   :r scripts/silver/proc_load_silver.sql
   EXEC silver.load_silver;
   ```

4. **Create the Gold views**

   ```sql
   :r scripts/gold/ddl_gold.sql
   ```

5. **Query the warehouse**

   ```sql
   SELECT * FROM gold.fact_sales;
   SELECT * FROM gold.dim_customers;
   SELECT * FROM gold.dim_products;
   ```

## Notes

- Bronze load and Silver transform steps are idempotent — tables are dropped/truncated and reloaded each run, so procedures can be re-executed safely.
- The Gold layer is implemented as **views**, not materialized tables, so it always reflects the latest Silver data.
- `tests/` and `docs/` are scaffolded for data quality checks and further documentation as the project grows.
