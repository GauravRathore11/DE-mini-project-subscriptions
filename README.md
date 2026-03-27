# Subscription Data Engineering Pipeline

## Overview

A production-grade data engineering project implementing a medallion architecture (Bronze → Silver) for subscription business analytics. This pipeline processes multi-source subscription data through Delta Lake tables, providing clean, validated datasets for downstream analytics and business intelligence.

## Architecture

### Medallion Architecture

```
Bronze Layer (Raw Data)
    ↓
Silver Layer (Cleaned & Transformed)
    • customers
    • products
    • opportunity
    • employees
    • countries
    • fx_rates
```

### Technology Stack

* **Platform**: Databricks (Azure)
* **Compute**: Serverless
* **Storage**: Azure Blob Storage / Delta Lake
* **Orchestration**: Delta Live Tables
* **Data Ingestion**: Fivetran
* **Processing**: Apache Spark (PySpark)

## Project Structure

```
DE-mini-project-subscriptions/
├── README.md
└── 02_silver/
    ├── customers.ipynb
    ├── products.ipynb
    ├── opportunity.ipynb
    ├── employees.ipynb
    ├── countries.ipynb
    ├── fx_rates.ipynb
    └── data_quality_report.ipynb
```

## Data Sources

### Customer Data
* Customer profiles and account information
* Account creation dates
* Customer internal IDs
* Deduplication and data cleaning

### Product Data
* Product catalog and metadata
* Product categorization
* Pricing information

### Opportunity Data
* Sales opportunities and pipeline
* Deal stages and values
* Opportunity tracking

### Supporting Dimensions
* **Employees**: Staff information for relationship management
* **Countries**: Geographic dimension for regional analysis
* **FX Rates**: Currency conversion for multi-currency operations

## Data Transformations

### Bronze to Silver Processing

1. **Metadata Removal**: Strips Fivetran ingestion metadata columns
   * `_file`
   * `_fivetran_synced`
   * `_modified`
   * `_line`

2. **Data Type Conversions**: Standardizes date formats and data types

3. **Data Enrichment**: Extracts derived fields (e.g., customer internal IDs)

4. **Deduplication**: Removes duplicate records to ensure data integrity

5. **Schema Validation**: Enforces consistent schemas across tables

## Data Quality Framework

### Quality Metrics

The `data_quality_report` notebook generates comprehensive quality metrics:

* **Completeness**: Null and missing value analysis
* **Uniqueness**: Distinct count validation
* **Consistency**: Data type and format verification
* **Coverage**: Row count tracking across all tables

### Monitored Tables

* `employees`
* `customers`
* `countries`
* `fx_rates`
* `products`
* `opportunity`

## Database Schema

### Bronze Schema
**Namespace**: `subscription_pipeline.bronze`

Contains raw, unprocessed data ingested via Fivetran from Azure Blob Storage with original structure and metadata.

### Silver Schema
**Namespace**: `subscription_pipeline.silver`

Contains cleaned, transformed, and business-ready tables:
* `subscription_pipeline.silver.customers`
* `subscription_pipeline.silver.products`
* `subscription_pipeline.silver.opportunity`
* `subscription_pipeline.silver.employees`
* `subscription_pipeline.silver.countries`
* `subscription_pipeline.silver.fx_rates`

## Pipeline Configuration

### Delta Live Tables Setup

1. **Compute**: Serverless (required)
2. **Target Schema**: `subscription_pipeline`
3. **Storage Location**: Delta Lake (managed tables)
4. **Notebooks**: All notebooks in `02_silver/` directory

### Execution Strategy

* **Mode**: Triggered or Continuous
* **Cluster Policy**: Serverless compute
* **Update Strategy**: Full refresh or incremental (configurable)

## Getting Started

### Prerequisites

* Databricks workspace on Azure with serverless compute enabled
* Azure Blob Storage account configured
* Unity Catalog access (if using governed tables)
* Fivetran connection configured for data ingestion from Azure Blob
* Source data available in Bronze layer

### Setup Instructions

1. **Clone/Import Project**
   ```
   Import all notebooks from the DE-mini-project-subscriptions folder
   ```

2. **Create Delta Live Tables Pipeline**
   * Navigate to: Workflows → Delta Live Tables
   * Click "Create Pipeline"
   * Select "Serverless" compute
   * Add notebook paths from `02_silver/` directory
   * Set target schema: `subscription_pipeline`

3. **Configure Source Data**
   * Ensure Fivetran is syncing data from Azure Blob Storage to Bronze layer
   * Verify Bronze tables exist: `subscription_pipeline.bronze.*`

4. **Run Pipeline**
   * Start the DLT pipeline
   * Monitor execution in the pipeline UI
   * Verify Silver tables are created

5. **Validate Data Quality**
   * Run the `data_quality_report` notebook
   * Review metrics for completeness and consistency

## Monitoring & Observability

### Pipeline Health

* Delta Live Tables UI provides real-time pipeline monitoring
* Track data lineage through the Databricks lineage view
* Monitor table metrics (row counts, update timestamps)

### Data Quality Monitoring

* Run data quality reports after each pipeline execution
* Set up alerts for unexpected null counts or row count changes
* Track distinct value counts for key business fields

## Maintenance

### Regular Tasks

* Monitor Fivetran sync status from Azure Blob Storage
* Review data quality reports weekly
* Validate schema changes from source systems
* Optimize Delta tables (OPTIMIZE, VACUUM)

### Troubleshooting

* **Pipeline Failures**: Check DLT event logs
* **Data Quality Issues**: Review quality report for anomalies
* **Missing Data**: Verify Fivetran sync completion from Azure Blob
* **Schema Changes**: Update transformation logic as needed

## Future Enhancements

* [ ] Gold layer for aggregated business metrics
* [ ] Automated data quality alerts
* [ ] Incremental processing optimization
* [ ] Advanced data validation rules
* [ ] Integration with BI tools (Tableau, Power BI)
* [ ] MLOps integration for predictive analytics

## Contributing

When adding new transformations:

1. Follow existing notebook structure
2. Include helper functions for reusability
3. Add data quality checks for new tables
4. Update this README with new components
5. Test transformations before deploying to production

## Support

For issues or questions:
* Review DLT pipeline logs
* Check Databricks workspace event logs
* Validate source data in Bronze layer
* Verify Fivetran connector status for Azure Blob Storage

---

**Project Owner**: gauravrathore856@gmail.com  
**Last Updated**: 2025  
**Version**: 1.0  
**Platform**: Databricks on Azure