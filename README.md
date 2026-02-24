# SQL Server Data Warehouse Implementation

A production-style Data Warehouse built with **SQL Server** integrating **CRM** and **ERP** source systems into an analytics-ready **Star Schema**, following the **Medallion Architecture** (Bronze → Silver → Gold).

---

## Architecture Overview

```
CRM Sources                     ERP Sources
(cust_info, prd_info,          (CUST_AZ12, LOC_A101,
 sales_details)                 PX_CAT_G1V2)
       │                               │
       └──────────────┬────────────────┘
                      ▼
              ┌──────────────┐
              │ BRONZE LAYER │  Raw ingestion via BULK INSERT
              │  (6 tables)  │  No transformation, audit trail
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ SILVER LAYER │  Cleansing, deduplication,
              │  (6 tables)  │  normalization, type casting
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │  GOLD LAYER  │  Star Schema views for analytics
              │  (3 views)   │  dim_customers, dim_products,
              └──────┬───────┘  fact_sales
                     │
                     ▼
              Power BI Dashboard
```

---

## Tech Stack

| Tool | Usage |
|------|-------|
| SQL Server / T-SQL | Database engine and query language |
| Stored Procedures | Automated ETL pipeline (`load_bronze`, `load_silver`) |
| BULK INSERT | High-performance CSV ingestion |
| Window Functions | Deduplication (ROW_NUMBER), date calculation (LEAD) |
| Star Schema | Dimensional modeling for OLAP analytics |
| Power BI | Dashboard and data visualization |

---

## Data Sources

### CRM System
| File | Rows | Description |
|------|------|-------------|
| cust_info.csv | 18,493 | Customer demographics and status |
| prd_info.csv | 397 | Product catalog with cost and lifecycle |
| sales_details.csv | 60,398 | Sales transactions with amounts and dates |

### ERP System
| File | Rows | Description |
|------|------|-------------|
| CUST_AZ12.csv | 18,483 | Customer birthdate and gender |
| LOC_A101.csv | 18,484 | Customer country/location mapping |
| PX_CAT_G1V2.csv | 36 | Product category hierarchy |

---

## ETL Pipeline

### Bronze Layer — Raw Ingestion
- Truncate-and-reload pattern for idempotent runs
- No transformations; data preserved exactly as in source
- Includes load duration logging per table
- Error handling via TRY-CATCH with detailed error output

### Silver Layer — Transformation & Cleansing

| Table | Key Transformations |
|-------|-------------------|
| crm_cust_info | TRIM whitespace, decode gender/marital status codes, deduplicate by latest record |
| crm_prd_info | Extract category ID from product key, decode product line codes, compute end dates with LEAD() |
| crm_sales_details | Validate and cast 8-digit date strings, recalculate invalid sales/price values |
| erp_cust_az12 | Strip 'NAS' prefix from IDs, nullify future birthdates, standardize gender strings |
| erp_loc_a101 | Remove hyphens from customer ID, map country codes (DE→Germany, US/USA→United States) |
| erp_px_cat_g1v2 | Pass-through (source data is clean) |

### Gold Layer — Star Schema

```
         dim_products ──────────────┐
         (product_key)              │
                                    ▼
dim_customers ──────────────► fact_sales
(customer_key)                (order_number,
                               sales_amount,
                               quantity, price,
                               order/ship/due dates)
```

**dim_customers** — Merged from CRM + ERP, includes surrogate key, country, gender (CRM-primary with ERP fallback), birthdate
**dim_products** — Merged from CRM + ERP categories, filtered to current active products only
**fact_sales** — 60K+ sales transactions enriched with dimension foreign keys

---

## How to Run

```sql
-- Step 1: Initialize database and schemas
-- Run: sql-server/scripts/bronze_layer/init_database.sql

-- Step 2: Create bronze tables
-- Run: sql-server/scripts/bronze_layer/bronze_layer_table_creation.sql

-- Step 3: Create silver tables
-- Run: sql-server/scripts/silver_layer/ddl_silver.sql

-- Step 4: Create gold views
-- Run: sql-server/scripts/gold_layer/ddl_gold.sql

-- Step 5: Load raw data into bronze
EXEC bronze.load_bronze;

-- Step 6: Transform and load into silver
EXEC silver.load_silver;

-- Step 7: Query analytics layer
SELECT * FROM gold.fact_sales;
SELECT * FROM gold.dim_customers;
SELECT * FROM gold.dim_products;
```

> **Note:** Update file paths in `load_bronze_stored_procedure.sql` to match your local CSV directory.

---

## Project Highlights

- Integrated **two heterogeneous source systems** (CRM + ERP) with different ID formats and conventions into a single unified model
- Implemented **13+ data quality rules** including deduplication, format standardization, business rule validation (sales = qty × price), and null handling
- Designed a fully normalized **Star Schema** enabling efficient OLAP queries across customers, products, and sales
- Built **idempotent stored procedures** (truncate-reload) safe to run repeatedly without side effects
- Achieved **referential integrity** across fact and dimension tables via surrogate keys generated with ROW_NUMBER()

---

## Dashboard Screenshots

### Executive Summary
![Executive Summary](image/sales-executive.png)

### Product Performance
![Product Analysis](image/product-perfotmance.png)

### Customer Insights
![Customer Insights](image/customer-insight.png)

### Sales Trends
![Sales Trends](image/sales-trends.png)
