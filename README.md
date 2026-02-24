# SQL Server Data Warehouse Implementation

A layered Data Warehouse built with **SQL Server** following the **Medallion Architecture** (Bronze → Silver → Gold).
Raw CRM and ERP data is ingested, cleansed, and exposed as a Star Schema for analytical reporting in Power BI.

---

## Architecture
![Architecture](image/diagram.drawio(1).png)
## Tech Stack
- **SQL Server** / T-SQL
- **Stored Procedures** (bronze.load_bronze, silver.load_silver)
- **Star Schema** — Dimensional Modeling
- **Power BI** — Dashboard & KPI Reporting

---

## Data Sources

| Source | File | Description |
|--------|------|-------------|
| CRM | cust_info.csv | Customer master data |
| CRM | prd_info.csv | Product catalog |
| CRM | sales_details.csv | Sales transactions |
| ERP | CUST_AZ12.csv | Customer demographics |
| ERP | LOC_A101.csv | Location / country mapping |
| ERP | PX_CAT_G1V2.csv | Product categories |

---

## Silver Layer Transformations

| Table | Transformations Applied |
|-------|------------------------|
| crm_cust_info | Whitespace trim, marital status decode (S/M → Single/Married), gender decode, deduplication by latest record |
| crm_prd_info | Category ID extraction from product key, product line decode (M/R/S/T), date cast, end date calculation via LEAD() |
| crm_sales_details | 8-digit integer → DATE conversion, sales = qty × price validation & correction |
| erp_cust_az12 | NAS prefix removal, future birthdate nullification, gender normalization |
| erp_loc_a101 | Hyphen removal from customer IDs, country code → full name (DE→Germany, US→United States) |
| erp_px_cat_g1v2 | Direct pass-through with audit column |

---

## Gold Layer — Star Schema
```
                    ┌────────────────┐
                    │ dim_customers  │
                    │ customer_key   │
                    │ customer_id    │
                    │ full_name      │
                    │ country        │
                    │ gender         │
                    │ birthdate      │
                    └───────┬────────┘
                            │
┌────────────────┐   ┌──────┴───────────┐
│ dim_products   │   │   fact_sales     │
│ product_key    ├───┤ order_number     │
│ product_name   │   │ product_key (FK) │
│ category       │   │ customer_key (FK)│
│ subcategory    │   │ order_date       │
│ product_line   │   │ sales_amount     │
│ product_cost   │   │ quantity         │
└────────────────┘   │ unit_price       │
                     └──────────────────┘
```

---

## Project Structure
```
SQLServer-DWH-Implementation/
├── datas/
│   ├── source_crm/          # CRM CSV source files
│   └── source_erp/          # ERP CSV source files
├── sql-server/scripts/
│   ├── bronze_layer/
│   │   ├── init_database.sql              # Creates DataWarehouse DB + schemas
│   │   ├── bronze_layer_table_creation.sql
│   │   └── load_bronze_stored_procedure.sql  # BULK INSERT loader
│   ├── silver_layer/
│   │   ├── ddl_silver.sql                 # Silver table definitions
│   │   └── silver_layer_created.sql       # Transformation stored procedure
│   └── gold_layer/
│       └── ddl_gold.sql                   # Star schema views
├── tests/                   # Data quality validation queries
├── image/                   # Dashboard screenshots
└── dashboard-visualization.pbix
```

---

## How to Run

### Prerequisites
- SQL Server 2019+ (or SQL Server 2022)
- Sufficient permissions to create databases, schemas, tables, and stored procedures
- CSV source files placed in: `C:\sql\dwh_project\datasets\` (adjust path in `load_bronze_stored_procedure.sql` if needed)

### Execution Order
```sql
-- Step 1: Initialize database and schemas
-- Run: sql-server/scripts/bronze_layer/init_database.sql

-- Step 2: Create Bronze tables
-- Run: sql-server/scripts/bronze_layer/bronze_layer_table_creation.sql

-- Step 3: Create Silver tables
-- Run: sql-server/scripts/silver_layer/ddl_silver.sql

-- Step 4: Create Gold views (Star Schema)
-- Run: sql-server/scripts/gold_layer/ddl_gold.sql

-- Step 5: Load data
EXEC bronze.load_bronze;
EXEC silver.load_silver;

-- Step 6: Query the Star Schema
SELECT * FROM gold.fact_sales;
SELECT * FROM gold.dim_customers;
SELECT * FROM gold.dim_products;
```

---

## Dashboard Screenshots

### Executive Summary
![Executive Summary](image/sales-executive.png)

### Product Analysis
![Product Analysis](image/product-perfotmance.png)

### Customer Insights
![Customer Insights](image/customer-insight.png)

### Sales Trends & Performance
![Sales Trends](image/sales-trends.png)
