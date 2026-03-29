# Subscription Analytics Data Pipeline

## Executive Summary

A production-grade, end-to-end data engineering solution for subscription business analytics, implementing a complete medallion architecture (Bronze → Silver → Gold) with dimensional modeling and a unified data cube for KPI reporting. The pipeline processes multi-source subscription data from Azure Data Lake through Fivetran ingestion, Delta Lake transformations, and culminates in a comprehensive analytics layer supporting 10+ business KPIs.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Data Flow](#data-flow)
3. [Technology Stack](#technology-stack)
4. [Data Sources](#data-sources)
5. [Pipeline Layers](#pipeline-layers)
6. [Dimensional Model](#dimensional-model)
7. [Data Cube](#data-cube)
8. [Key Performance Indicators (KPIs)](#key-performance-indicators-kpis)
9. [Project Structure](#project-structure)
10. [Setup Instructions](#setup-instructions)
11. [Data Quality](#data-quality)
12. [Monitoring & Maintenance](#monitoring--maintenance)
13. [Future Enhancements](#future-enhancements)

---

## Architecture Overview

### End-to-End Data Pipeline

```
Azure Data Lake (Source)
    ↓
Azure Blob Storage (Staging)
    ↓
Fivetran Connector (Ingestion)
    ↓
Bronze Layer (Raw Delta Tables)
    ↓
Silver Layer (Cleaned & Validated)
    ↓
Gold Layer (Dimensional Model)
    ├── Fact Tables
    └── Dimension Tables
    ↓
Data Cube (Unified Analytics Dataset)
    ↓
KPI Layer (Business Metrics)
```

### Medallion Architecture Pattern

* **Bronze Layer**: Raw data ingested from Azure Blob Storage via Fivetran with original structure and metadata
* **Silver Layer**: Cleaned, validated, and standardized business-ready tables
* **Gold Layer**: Dimensional model (star schema) with fact and dimension tables optimized for analytics
* **Data Cube**: Denormalized, pre-joined dataset for high-performance KPI calculations

---

## Data Flow

### 1. Source: Azure Data Lake
* Primary data repository for subscription business data
* Contains customer, product, opportunity, employee, geographic, and financial data
* Raw CSV/JSON files organized by domain

### 2. Staging: Azure Blob Storage
* Intermediate storage layer for data transfer
* Organized into structured folders by data entity
* Optimized for Fivetran connector access

### 3. Ingestion: Fivetran
* **Connector Type**: Azure Blob Storage
* **Sync Mode**: Incremental with change data capture
* **Target**: Databricks Delta Lake (Bronze schema)
* **Metadata Tracking**: Adds Fivetran audit columns:
  * `_file`: Source file path
  * `_fivetran_synced`: Sync timestamp
  * `_modified`: Last modification timestamp
  * `_line`: Line number in source file

### 4. Bronze Layer Processing
* Raw data loaded as Delta tables in `subscription_pipeline.bronze` schema
* Preserves all source data and Fivetran metadata
* No transformations applied—maintains data lineage

### 5. Silver Layer Processing
* Reads from Bronze tables
* Removes Fivetran metadata columns
* Applies data quality rules:
  * Data type standardization
  * Date format normalization
  * Deduplication logic
  * Null handling
* Writes to `subscription_pipeline.silver` schema

### 6. Gold Layer Processing
* Implements star schema dimensional model
* Creates fact tables with business metrics
* Creates dimension tables for analysis attributes
* Writes to `subscription_pipeline.gold` schema

### 7. Data Cube Creation
* Joins all fact and dimension tables into a unified dataset
* Pre-calculates derived metrics
* Stored as `subscription_pipeline.gold.data_cube`
* Enables single-query KPI calculations

### 8. KPI Calculations
* Reads from unified data cube
* Calculates 10+ business KPIs
* Supports multi-dimensional analysis

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|----------|
| **Cloud Platform** | Azure | Infrastructure and storage |
| **Data Lake** | Azure Data Lake Storage Gen2 | Primary data repository |
| **Blob Storage** | Azure Blob Storage | Staging area for Fivetran |
| **Data Ingestion** | Fivetran | Automated data connector |
| **Analytics Platform** | Databricks | Data processing and transformation |
| **Compute** | Serverless | Auto-scaling compute resources |
| **Storage Format** | Delta Lake | ACID-compliant table format |
| **Processing Engine** | Apache Spark (PySpark) | Distributed data processing |
| **Orchestration** | Delta Live Tables (DLT) | Pipeline automation |
| **Catalog** | Unity Catalog | Data governance and lineage |

---

## Data Sources

### Customer Data (`customers`)
* Customer profiles and account information
* Account creation dates and customer lifecycle
* Customer internal IDs and classification
* Industry type segmentation

### Product Data (`products`)
* Product catalog and metadata
* Product categorization and hierarchies
* Pricing information and billing cycles
* Active status tracking

### Opportunity Data (`opportunity`)
* Sales opportunities and subscription pipeline
* Deal stages, close dates, and values
* Subscription start and end dates
* Revenue amounts in GBP

### Employee Data (`employees`)
* Staff information for customer relationship management
* Employee roles and responsibilities
* Regional assignments
* Active status flags

### Geographic Data (`countries`)
* Country master data
* Country codes and names
* Currency codes for multi-currency support

### FX Rates (`fx_rates`)
* Foreign exchange rates for currency conversion
* Historical rate tracking
* Multi-currency revenue normalization

---

## Pipeline Layers

### Bronze Layer (`subscription_pipeline.bronze`)

**Purpose**: Raw data ingestion from Fivetran

**Tables**:
* `subscription_pipeline.bronze.customers`
* `subscription_pipeline.bronze.products`
* `subscription_pipeline.bronze.opportunity`
* `subscription_pipeline.bronze.employees`
* `subscription_pipeline.bronze.country_master`
* `subscription_pipeline.bronze.fx_rates`

**Characteristics**:
* Contains all Fivetran metadata columns
* No data transformations
* Preserves original data types
* Incremental updates from Fivetran

### Silver Layer (`subscription_pipeline.silver`)

**Purpose**: Cleaned and validated business-ready data

**Tables**:
* `subscription_pipeline.silver.customers`
* `subscription_pipeline.silver.products`
* `subscription_pipeline.silver.opportunity`
* `subscription_pipeline.silver.employees`
* `subscription_pipeline.silver.countries`
* `subscription_pipeline.silver.fx_rates`

**Transformations**:
1. **Metadata Removal**: Strips Fivetran audit columns
2. **Data Type Conversions**: Standardizes date formats and data types
3. **Data Enrichment**: Extracts derived fields (e.g., customer internal IDs)
4. **Deduplication**: Removes duplicate records
5. **Schema Validation**: Enforces consistent schemas

**Notebooks**:
* `02_silver/customers.ipynb`
* `02_silver/products.ipynb`
* `02_silver/opportunity.ipynb`
* `02_silver/employees.ipynb`
* `02_silver/countries.ipynb`
* `02_silver/fx_rates.ipynb`
* `02_silver/data_quality_report.ipynb`

### Gold Layer (`subscription_pipeline.gold`)

**Purpose**: Dimensional model for analytics

**Schema**: Star schema with fact and dimension tables

---

## Dimensional Model

### Star Schema Design

```
        dim_date          dim_customer
            ↓                   ↓
            ↓                   ↓
dim_employee → fact_subscription ← dim_product
                      ↓
                dim_country
```

### Fact Table

#### `fact_subscription`
**Grain**: One row per subscription (opportunity)

**Metrics**:
* `revenue_gbp`: Revenue in GBP
* `prev_revenue`: Previous period revenue
* `revenue_change`: Change in revenue

**Flags**:
* `is_new_customer`: First subscription indicator
* `is_upgrade`: Revenue increase indicator
* `is_downgrade`: Revenue decrease indicator

**Foreign Keys**:
* `customer_id`
* `product_id`
* `start_date` (links to dim_date)
* `end_date` (links to dim_date)

**Source Notebook**: `03_gold/facts/fact_opportunity`

### Dimension Tables

#### `dim_customer`
**Attributes**:
* `customer_id` (PK)
* `customer_name`
* `country`
* `industry_type`
* `account_created_date`
* `is_active`
* `customer_type` (active/inactive)

**Source Notebook**: `03_gold/dimensions/dim_customer.ipynb`

#### `dim_product`
**Attributes**:
* `product_id` (PK)
* `product_name`
* `plan_name`
* `billing_cycle`
* `list_price`
* `is_active`

**Source Notebook**: `03_gold/dimensions/dim_product.ipynb`

#### `dim_employee`
**Attributes**:
* `employee_id` (PK)
* `employee_name`
* `role`
* `region`
* `is_active_flag`

**Source Notebook**: `03_gold/dimensions/dim_employee.ipynb`

#### `dim_country`
**Attributes**:
* `country_code` (PK)
* `country_name`
* `currency_code`

**Source Notebook**: `03_gold/dimensions/dim_country`

#### `dim_date`
**Attributes**:
* `date` (PK)
* `year`
* `month`
* `day`
* `quarter`
* `week`
* `year_month`
* `financial_year` (April-March fiscal year)

**Source Notebook**: `03_gold/dimensions/dim_date`

---

## Data Cube

### Purpose
A **unified, denormalized dataset** that pre-joins all fact and dimension tables, enabling high-performance KPI calculations without repeated joins.

### Table: `subscription_pipeline.gold.data_cube`

**Schema**: 33 columns combining:
* All fact table metrics
* All dimension table attributes
* Computed metrics:
  * `subscription_duration_days`
  * `is_active_subscription`
  * `customer_lifetime_months`
  * `revenue_per_day`
  * `is_churned`

**Row Count**: ~150,000 subscription records

**Source Notebook**: `03_gold/data_cube/build_data_cube`

**Export Formats**:
* Delta table: `subscription_pipeline.gold.data_cube`
* CSV: `/Workspace/.../03_gold/data_cube/data_cube.csv`
* Excel: `/Workspace/.../03_gold/data_cube/data_cube.xlsx`

### Benefits
1. **Single Query Access**: All KPIs from one table
2. **Performance**: No runtime joins required
3. **Consistency**: Same data source for all metrics
4. **Simplicity**: Easy for analysts and BI tools

---

## Key Performance Indicators (KPIs)

### Source Notebook: `03_gold/kpis/kpi`

All KPIs are calculated from the unified data cube (`subscription_pipeline.gold.data_cube`).

### Available KPIs

1. **Customer Gains and Losses**
   * New customers acquired per period
   * Customers lost (churned) per period
   * Net customer change

2. **Customer Loss Trend Analysis**
   * Period-over-period churn comparison
   * Percentage change in customer losses
   * Churn acceleration/deceleration

3. **Highest-Value Customers**
   * Top customers by total recurring revenue
   * Total subscription count per customer
   * Customer lifetime analysis

4. **Customer Expansion Analysis**
   * Customers with product usage increases
   * Total upgrades per customer
   * Average revenue change

5. **Customer Contraction Analysis**
   * Customers reducing product usage
   * Total downgrades per customer
   * At-risk customer identification

6. **Revenue Retention (Current FY)**
   * Recurring revenue from existing customers
   * Revenue retention rate
   * Financial year comparison

7. **Revenue Growth Decomposition (Current FY)**
   * Revenue from upgrades vs. downgrades
   * Net revenue change
   * Growth attribution

8. **Product Performance**
   * Top products by subscription count
   * Revenue by product
   * Product adoption rates

9. **Regional Performance**
   * Revenue by country/region
   * Customer distribution by geography
   * Regional growth trends

10. **Employee Performance**
    * Subscriptions by account manager
    * Revenue by employee
    * Team productivity metrics

---

## Project Structure

```
DE-mini-project-subscriptions/
├── README.md                          # This file
│
├── 02_silver/                         # Silver Layer Transformations
│   ├── customers.ipynb                # Customer data cleaning
│   ├── products.ipynb                 # Product data cleaning
│   ├── opportunity.ipynb              # Opportunity data cleaning
│   ├── employees.ipynb                # Employee data cleaning
│   ├── countries.ipynb                # Country data cleaning
│   ├── fx_rates.ipynb                 # FX rates data cleaning
│   └── data_quality_report.ipynb      # Data quality metrics
│
├── 03_gold/                           # Gold Layer Analytics
│   ├── dimensions/                    # Dimension Tables
│   │   ├── dim_customer.ipynb         # Customer dimension
│   │   ├── dim_product.ipynb          # Product dimension
│   │   ├── dim_employee.ipynb         # Employee dimension
│   │   ├── dim_country                # Country dimension
│   │   └── dim_date                   # Date dimension
│   │
│   ├── facts/                         # Fact Tables
│   │   └── fact_opportunity           # Subscription fact table
│   │
│   ├── data_cube/                     # Unified Data Cube
│   │   ├── build_data_cube            # Data cube creation
│   │   ├── data_cube.csv              # CSV export
│   │   └── data_cube.xlsx             # Excel export
│   │
│   └── kpis/                          # KPI Calculations
│       └── kpi                        # All business KPIs
```

---

## Setup Instructions

### Prerequisites

* Azure subscription with the following services:
  * Azure Data Lake Storage Gen2
  * Azure Blob Storage
* Databricks workspace on Azure
  * Serverless compute enabled
  * Unity Catalog configured
* Fivetran account with Azure Blob Storage connector
* Source data available in Azure Data Lake

### Step 1: Configure Azure Storage

1. **Set up Azure Data Lake**:
   ```bash
   # Create storage account
   az storage account create \
     --name <storage-account-name> \
     --resource-group <resource-group> \
     --location <region> \
     --sku Standard_LRS \
     --kind StorageV2 \
     --hierarchical-namespace true
   ```

2. **Create Blob Storage container**:
   ```bash
   az storage container create \
     --name subscription-data \
     --account-name <storage-account-name>
   ```

3. **Upload source data** from Data Lake to Blob Storage:
   * Organize by entity (customers, products, opportunities, etc.)
   * Maintain consistent file naming conventions
   * Use CSV or JSON format

### Step 2: Configure Fivetran

1. **Create Fivetran connector**:
   * Connector type: Azure Blob Storage
   * Authentication: SAS token or Storage Account Key
   * Schema: `subscription_pipeline.bronze`

2. **Configure sync settings**:
   * Sync mode: Incremental
   * Sync frequency: Every 6 hours (or as required)
   * Target: Databricks workspace

3. **Map source files to tables**:
   * `customers.csv` → `subscription_pipeline.bronze.customers`
   * `products.csv` → `subscription_pipeline.bronze.products`
   * `opportunities.csv` → `subscription_pipeline.bronze.opportunity`
   * `employees.csv` → `subscription_pipeline.bronze.employees`
   * `countries.csv` → `subscription_pipeline.bronze.country_master`
   * `fx_rates.csv` → `subscription_pipeline.bronze.fx_rates`

4. **Run initial sync** to populate Bronze tables

### Step 3: Set Up Databricks

1. **Import project notebooks**:
   ```bash
   # Clone this repository or import notebooks manually
   git clone <repository-url>
   ```

2. **Create Unity Catalog schema**:
   ```sql
   CREATE SCHEMA IF NOT EXISTS subscription_pipeline.bronze;
   CREATE SCHEMA IF NOT EXISTS subscription_pipeline.silver;
   CREATE SCHEMA IF NOT EXISTS subscription_pipeline.gold;
   ```

3. **Verify Bronze tables** exist after Fivetran sync:
   ```sql
   SHOW TABLES IN subscription_pipeline.bronze;
   ```

### Step 4: Create Delta Live Tables Pipeline

1. **Navigate to Workflows** → **Delta Live Tables**

2. **Create new pipeline** with the following settings:
   * **Name**: `subscription_analytics_pipeline`
   * **Product Edition**: Advanced
   * **Compute**: Serverless
   * **Target Schema**: `subscription_pipeline`
   * **Storage Location**: (use default or specify)

3. **Add notebook paths**:
   * Silver layer notebooks: `02_silver/*`
   * Gold layer notebooks: `03_gold/dimensions/*`, `03_gold/facts/*`
   * Data cube: `03_gold/data_cube/build_data_cube`

4. **Configure pipeline settings**:
   * Development mode: Enabled (for testing)
   * Channel: Current
   * Auto Optimization: Enabled

5. **Start pipeline** and monitor execution

### Step 5: Validate Pipeline

1. **Check Silver tables**:
   ```sql
   SELECT COUNT(*) FROM subscription_pipeline.silver.customers;
   SELECT COUNT(*) FROM subscription_pipeline.silver.opportunity;
   ```

2. **Check Gold tables**:
   ```sql
   SELECT COUNT(*) FROM subscription_pipeline.gold.dim_customer;
   SELECT COUNT(*) FROM subscription_pipeline.gold.fact_subscription;
   ```

3. **Verify Data Cube**:
   ```sql
   SELECT COUNT(*), COUNT(DISTINCT customer_id) 
   FROM subscription_pipeline.gold.data_cube;
   ```

4. **Run data quality report**:
   * Execute `02_silver/data_quality_report.ipynb`
   * Review null counts and distinct values

### Step 6: Generate KPIs

1. **Run KPI notebook**: `03_gold/kpis/kpi`
2. **Review KPI outputs** for each business metric
3. **Export results** to CSV or dashboard

---

## Data Quality

### Quality Framework

The `data_quality_report` notebook (`02_silver/data_quality_report.ipynb`) provides comprehensive quality metrics:

#### Metrics Tracked
1. **Completeness**: Null and missing value counts
2. **Uniqueness**: Distinct value counts for each column
3. **Consistency**: Data type validation
4. **Coverage**: Row count tracking across all tables

#### Monitored Tables
* `subscription_pipeline.silver.employees`
* `subscription_pipeline.silver.customers`
* `subscription_pipeline.silver.countries`
* `subscription_pipeline.silver.fx_rates`
* `subscription_pipeline.silver.products`
* `subscription_pipeline.silver.opportunity`

### Quality Checks

#### Silver Layer Validations
* No duplicate records based on primary keys
* Date formats standardized (YYYY-MM-DD)
* Required fields (customer_id, product_id) are non-null
* Referential integrity for foreign keys

#### Gold Layer Validations
* Dimension tables have unique primary keys
* Fact table foreign keys exist in dimension tables
* Revenue amounts are positive
* Date ranges are valid (start_date <= end_date)

---

## Monitoring & Maintenance

### Pipeline Health Monitoring

1. **Delta Live Tables Dashboard**:
   * Real-time pipeline execution status
   * Table refresh times
   * Data lineage visualization
   * Error logs and event history

2. **Fivetran Connector Monitoring**:
   * Sync status and frequency
   * Row counts synced
   * Sync errors and warnings
   * Azure Blob Storage connection health

3. **Table Metrics**:
   ```sql
   -- Monitor table sizes
   DESCRIBE DETAIL subscription_pipeline.silver.opportunity;
   
   -- Check last update times
   SELECT MAX(_fivetran_synced) FROM subscription_pipeline.bronze.customers;
   ```

### Regular Maintenance Tasks

#### Daily
* Monitor Fivetran sync status
* Check DLT pipeline execution logs
* Validate row counts in Silver and Gold layers

#### Weekly
* Run data quality reports
* Review KPI trends for anomalies
* Check Delta table versions and time travel

#### Monthly
* Optimize Delta tables:
  ```sql
  OPTIMIZE subscription_pipeline.silver.opportunity;
  OPTIMIZE subscription_pipeline.gold.fact_subscription;
  ```
* Vacuum old versions:
  ```sql
  VACUUM subscription_pipeline.silver.opportunity RETAIN 168 HOURS;
  ```
* Review storage costs and usage
* Validate schema changes from source systems

### Troubleshooting

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| **Fivetran Sync Failure** | Check Fivetran dashboard logs | Verify Azure Blob access credentials; check file formats |
| **Bronze Table Missing** | Fivetran connector not running | Re-run Fivetran sync; verify connector configuration |
| **Silver Transformation Error** | Review DLT event logs | Check for schema changes; validate data types |
| **Gold Table Join Issues** | Missing foreign key references | Run data quality report; fix referential integrity |
| **Data Cube Row Count Mismatch** | Dimension tables incomplete | Rebuild dimension tables; re-run data cube notebook |
| **KPI Calculation Errors** | Data cube schema changed | Verify data cube columns; update KPI queries |

---

## Future Enhancements

### Phase 1: Advanced Analytics
- [ ] Implement slowly changing dimensions (SCD Type 2) for dim_customer and dim_product
- [ ] Add time-series forecasting for revenue predictions
- [ ] Customer churn prediction using ML models
- [ ] Product recommendation engine

### Phase 2: Automation & Alerts
- [ ] Automated data quality alerts via email/Slack
- [ ] Anomaly detection for KPI thresholds
- [ ] Automated pipeline failure notifications
- [ ] Self-healing pipelines with retry logic

### Phase 3: Performance Optimization
- [ ] Incremental processing for fact table updates
- [ ] Partitioning strategy for large tables
- [ ] Z-ordering on frequently filtered columns
- [ ] Liquid clustering for optimal query performance

### Phase 4: Advanced Features
- [ ] Real-time streaming for opportunity updates
- [ ] Advanced data validation rules (Great Expectations)
- [ ] A/B testing framework for product experiments
- [ ] Customer segmentation models (RFM analysis)

### Phase 5: Integration & Visualization
- [ ] Power BI dashboard integration
- [ ] Tableau workbooks for executive reporting
- [ ] REST API for KPI consumption
- [ ] Slack bot for KPI queries
- [ ] Email reports with automated insights

### Phase 6: MLOps & AI
- [ ] Customer lifetime value (CLV) prediction models
- [ ] Next best action recommendations
- [ ] Automated insights generation using LLMs
- [ ] Integration with Databricks ML models

---

## Contributing

When adding new transformations or features:

1. **Follow Existing Patterns**:
   * Use consistent naming conventions
   * Follow medallion architecture principles
   * Add helper functions for reusability

2. **Documentation**:
   * Add cell titles to notebook cells
   * Include comments for complex logic
   * Update this README with new components

3. **Testing**:
   * Test transformations in development mode
   * Validate data quality after changes
   * Run end-to-end pipeline before deploying

4. **Code Quality**:
   * Use PySpark best practices
   * Avoid Spark actions in transformations (use lazy evaluation)
   * Optimize joins and filters
   * Add data quality checks for new tables

---

## Support & Contact

For issues, questions, or contributions:

1. **Pipeline Issues**:
   * Review DLT pipeline event logs
   * Check Databricks workspace event logs
   * Validate source data in Bronze layer

2. **Data Issues**:
   * Run data quality report notebook
   * Check Fivetran sync logs
   * Verify Azure Blob Storage file formats

3. **Performance Issues**:
   * Review query execution plans
   * Check table sizes with `DESCRIBE DETAIL`
   * Consider optimizing and vacuuming tables

**Project Owner**: gauravrathore856@gmail.com

---

## Appendix

### Key Concepts

**Medallion Architecture**: A data design pattern that organizes data into three layers (Bronze, Silver, Gold) with increasing levels of quality and refinement.

**Delta Lake**: An open-source storage layer that brings ACID transactions to data lakes, enabling reliable data pipelines.

**Data Cube**: A denormalized dataset that pre-joins multiple tables to enable fast analytical queries without runtime joins.

**Star Schema**: A dimensional modeling approach with a central fact table connected to multiple dimension tables.

**Fivetran**: A data integration platform that automates data pipelines from sources to destinations.

### Glossary

* **DLT**: Delta Live Tables - Databricks' declarative framework for building data pipelines
* **FY**: Financial Year (April - March)
* **GBP**: British Pound Sterling - base currency for revenue
* **KPI**: Key Performance Indicator - quantifiable measure of business performance
* **OLAP**: Online Analytical Processing - analytical query processing
* **PK**: Primary Key - unique identifier for a table row
* **SCD**: Slowly Changing Dimension - technique for tracking historical changes in dimensions

---

**Version**: 2.0  
**Last Updated**: January 2025  
**Platform**: Databricks on Azure  
**License**: Internal Use Only