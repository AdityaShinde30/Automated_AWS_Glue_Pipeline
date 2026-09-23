# End-to-End AWS Glue Data Pipeline Using Medallion Architecture

## 📌 Project Overview

This project implements an end-to-end **ETL/ELT data pipeline on AWS Cloud** using the **Medallion Architecture**. The pipeline processes raw source data through **Bronze, Silver, and Gold layers**, applying data ingestion, transformation, quality validation, data cataloging, and analytical processing at each stage.

The pipeline is built using **Amazon S3, AWS Glue, AWS Glue Crawlers, AWS Glue Workflows, AWS Glue Triggers, AWS Glue Data Catalog, IAM Roles, Amazon Athena, and Unity Catalog**.

The entire pipeline can be orchestrated through an AWS Glue Workflow and executed on demand.

---

## 🏗️ Architecture

```text
                    ┌──────────────────────┐
                    │     Source Data      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      Amazon S3       │
                    │      RAW Layer       │
                    └──────────┬───────────┘
                               │
                        AWS Glue Job
                     raw_to_bronze.py
                               │
                               ▼
                    ┌──────────────────────┐
                    │     BRONZE Layer     │
                    │      Parquet         │
                    └──────────┬───────────┘
                               │
                         Glue Crawler
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Glue Data Catalog /  │
                    │    Unity Catalog     │
                    └──────────┬───────────┘
                               │
                               ▼
                         AWS Glue Job
                    bronze_to_silver.py
                               │
                               ▼
                    ┌──────────────────────┐
                    │      SILVER Layer    │
                    │ Curated + Rejected   │
                    └──────────┬───────────┘
                               │
                         Glue Crawler
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Glue Data Catalog /  │
                    │    Unity Catalog     │
                    └──────────┬───────────┘
                               │
                               ▼
                         AWS Glue Job
                     silver_to_gold.py
                               │
                               ▼
                    ┌──────────────────────┐
                    │       GOLD Layer     │
                    │     BI Reports       │
                    └──────────┬───────────┘
                               │
                         Glue Crawler
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Amazon Athena      │
                    │ Ad-hoc SQL Analytics │
                    └──────────────────────┘
```

---

# 1. Create the S3 Bucket

The first step is to create an **Amazon S3 bucket** that will act as the data lake storage layer.

The project follows the Medallion Architecture by maintaining separate folders for:

* `raw`
* `bronze`
* `silver`
* `gold`

Additional folder structures required by the pipeline can be created using the commands provided in the repository under:

```text
prepare_s3_structure
```

### Raw Layer

The assumption for this project is that source data is available in the `raw` folder, organized according to the required categories and dates.

For this implementation, the corresponding source files should be placed under their respective category folders inside the raw layer.

Example:

```text
s3://<bucket-name>/
│
├── raw/
│   ├── orders/
│   │   └── <date>/
│   ├── products/
│       └── <date>/
│
├── bronze/
├── silver/
└── gold/
```

---

# 2. Create IAM Role for AWS Glue

Create an IAM role that allows AWS Glue to access the required AWS resources.

The Glue execution role should have appropriate permissions for:

* Amazon S3
* AWS Glue
* Glue Data Catalog
* CloudWatch Logs
* Required catalog/governance services

This role will be attached to the AWS Glue ETL jobs and crawlers.

---

# 3. Raw-to-Bronze ETL Job

The first ETL stage converts raw source data into the **Bronze layer**.

Create an AWS Glue ETL job and attach the previously created Glue IAM role.

The transformation logic is available in:

```text
raw_to_bronze.py
```

### Processing Flow

```text
S3 Raw
   ↓
AWS Glue ETL
   ↓
Bronze
```

The job reads source data from the raw layer and stores the processed data in **Parquet format** in the Bronze layer.

### Metadata Columns

During ingestion, the pipeline adds operational metadata columns:

* `source_file_name`
* `ingestion_timestamp`
* `pipeline_run_id`

These columns provide **data lineage and traceability**.

For example, if a data quality issue is identified later, `source_file_name` and `pipeline_run_id` can be used to trace the problematic records back to the corresponding source file and pipeline execution.

---

# 4. Execute the Raw-to-Bronze Job

After configuring the Glue job:

1. Configure the S3 input path.
2. Configure the Bronze output path.
3. Configure the Glue IAM role.
4. Add the required job parameters.
5. Save the job.
6. Execute the job.

The processed data will be written to the Bronze S3 location in **Parquet format**.

---

# 5. Create AWS Glue Database

Create a database in the **AWS Glue Data Catalog**.

This database acts as the logical container for the tables generated by the Glue Crawlers.

Example:

```text
Database
   │
   ├── bronze_orders
   ├── bronze_products
   ├── silver_orders
   ├── silver_rejected_orders
   ├── gold_daily_product_sales
   └── gold_category_sales
```

---

# 6. Create Bronze Glue Crawler

Create an AWS Glue Crawler to discover the schema of the Bronze data.

Configure the crawler to point to the Bronze S3 location.

The crawler automatically:

* Discovers the schema
* Identifies data types
* Creates or updates catalog tables
* Registers the metadata in the Glue Data Catalog

The cataloged tables can then be accessed through analytical services such as Amazon Athena.

---

# 7. Query Bronze Data Using Amazon Athena

After the Bronze crawler successfully completes, open **Amazon Athena** and connect it to the Glue Data Catalog database.

You can now execute SQL queries directly against the Bronze tables.

Example:

```sql
SELECT *
FROM bronze_orders
LIMIT 10;
```

Athena provides a serverless SQL interface for performing ad-hoc analysis without provisioning database infrastructure.

---

# 8. Bronze-to-Silver ETL Job

The next stage transforms Bronze data into the **Silver layer**.

The transformation logic is available in:

```text
bronze_to_silver.py
```

Create another AWS Glue ETL job and configure the required IAM role and parameters.

### Processing Flow

```text
Bronze
   ↓
Data Quality Checks
   ↓
┌─────────────────────┐
│                     │
▼                     ▼
Curated Records    Rejected Records
│                     │
▼                     ▼
Silver Curated     Silver Rejected
```

This stage performs **data quality validation and cleansing**.

Records that satisfy the defined quality rules are written to the curated Silver dataset.

Records that fail validation are separated and written to a rejected dataset.

For example:

```text
silver/
├── curated/
│   └── orders/
│
└── rejected/
    └── orders/
```

This approach prevents invalid records from contaminating downstream analytical datasets while preserving them for investigation and remediation.

---

# 9. Create Silver Glue Crawler

After the Bronze-to-Silver job completes, create a Glue Crawler for the Silver S3 location.

The crawler discovers and registers the Silver datasets in the Glue Data Catalog.

Example tables:

```text
silver_orders
silver_rejected_orders
```

The catalog metadata can then be queried through Amazon Athena.

---

# 10. Query Silver Data Using Athena

Connect Athena to the Glue database and perform SQL-based analysis on the Silver tables.

The Silver layer contains cleaned and quality-validated data suitable for downstream analytical processing.

---

# 11. Silver-to-Gold ETL Job

The final transformation stage creates business-level analytical datasets in the **Gold layer**.

The transformation logic is available in:

```text
silver_to_gold.py
```

The Gold layer contains aggregated datasets designed for reporting and analytics.

### Gold Reports

The pipeline generates reports such as:

### 1. Daily Product Sales

Metrics include:

* `total_order`
* `total_quantity`
* `total_sales`

### 2. Category-Wise Sales

Metrics include:

* `total_order`
* `total_quantity`
* `total_sales`

Example:

```text
gold/
├── daily_product_sales/
│
└── category_sales/
```

These datasets are optimized for downstream analytics and reporting.

---

# 12. Create Gold Glue Crawler

Create a Glue Crawler pointing to the Gold S3 location.

The crawler discovers the Gold datasets and registers the corresponding tables in the catalog.

Example:

```text
gold_daily_product_sales
gold_category_sales
```

These tables can then be queried using Athena.

---

# 13. Amazon Athena Analytics

Once the Gold crawler completes, Athena can be used to query the final analytical datasets.

Example:

```sql
SELECT *
FROM gold_daily_product_sales
ORDER BY sales_date;
```

The Gold layer provides business-ready datasets that can be consumed for reporting, dashboards, and analytical use cases.

---

# 14. Automating the Pipeline with AWS Glue Workflow

After validating each individual component, the complete pipeline is automated using an **AWS Glue Workflow**.

The workflow connects the ETL jobs, crawlers, and triggers into a dependency-driven DAG.

### Workflow Execution

```text
Start
  │
  ▼
raw_to_bronze
  │
  ▼
Bronze Crawler
  │
  ▼
bronze_to_silver
  │
  ▼
Silver Crawler
  │
  ▼
silver_to_gold
  │
  ▼
Gold Crawler
  │
  ▼
End
```

---

# 15. Configure Glue Triggers

The workflow starts with an **on-demand trigger**.

The first trigger launches:

```text
raw_to_bronze
```

After the job completes successfully, the next trigger starts the Bronze crawler.

The subsequent components are connected using dependency-based triggers.

For example:

```text
Trigger
   ↓
raw_to_bronze
   ↓
Trigger: After Job Completion
   ↓
Bronze Crawler
   ↓
Trigger: After Crawler Completion
   ↓
bronze_to_silver
   ↓
Silver Crawler
   ↓
silver_to_gold
   ↓
Gold Crawler
```

This creates a fully orchestrated DAG where each component executes based on the successful completion of its upstream dependency.

---

# 16. End-to-End Workflow

The final automated pipeline can be summarized as:

```text
                ┌──────────────┐
                │  Source Data │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │   S3 RAW     │
                └──────┬───────┘
                       ▼
              raw_to_bronze.py
                       ▼
                ┌──────────────┐
                │ S3 BRONZE    │
                │   Parquet    │
                └──────┬───────┘
                       ▼
                 Glue Crawler
                       ▼
                Glue Data Catalog
                       ▼
             bronze_to_silver.py
                       ▼
                ┌──────────────┐
                │ S3 SILVER    │
                │ Curated      │
                │ Rejected     │
                └──────┬───────┘
                       ▼
                 Glue Crawler
                       ▼
                Glue Data Catalog
                       ▼
              silver_to_gold.py
                       ▼
                ┌──────────────┐
                │  S3 GOLD     │
                │   Reports    │
                └──────┬───────┘
                       ▼
                 Glue Crawler
                       ▼
                Glue Data Catalog
                       ▼
                 Amazon Athena
                       ▼
               SQL Analytics
```

---

# 17. Key Features

* End-to-end AWS data engineering pipeline
* Medallion Architecture: **Bronze, Silver, Gold**
* Amazon S3-based data lake
* AWS Glue ETL processing
* Glue Crawlers for automated schema discovery
* Glue Data Catalog for metadata management
* Glue Workflows for pipeline orchestration
* Glue Triggers for dependency management
* Data quality validation and rejected-record handling
* Parquet-based storage
* Data lineage and operational metadata
* IAM-based access control
* Amazon Athena for serverless SQL analytics
* Business-level Gold aggregations
* On-demand end-to-end pipeline execution

---

# 18. Repository Structure

```text
.
├── prepare_s3_structure/
│   
├── layes_wise_schema/
|
├── raw_to_bronze.py
├── bronze_to_silver.py
├── silver_to_gold.py
│
└── README.md
```

---

## 🎯 Project Outcome

This project demonstrates how to build a production-oriented **AWS data lake ETL/ELT pipeline** using the Medallion Architecture. It covers the complete data lifecycle, from raw data ingestion and transformation to data quality validation, cataloging, orchestration, and business-level analytics.

The combination of **Amazon S3, AWS Glue, Glue Data Catalog, Glue Workflows, IAM, Unity Catalog, and Amazon Athena** provides a scalable foundation for implementing cloud-based data engineering workloads.
