# Finance Data Pipeline Demo

## Overview
This project demonstrates an end-to-end **AWS S3 → Snowflake ETL pipeline** for financial transaction data.  
It covers:

- Raw data ingestion
- Staging table creation and cleaning
- Fact and dimension table creation
- Incremental load automation using **STREAM** and **TASK**
- Application of business rules
- Analytics view

---

## Architecture Diagram

```text
+----------------+       +-------------------+       +--------------------+
|   AWS S3       | ----> | Snowflake RAW      | ----> | Staging Table      |
| (CSV uploads)  |       | Table (RAW)       |       | STG_TRANSACTIONS   |
+----------------+       +-------------------+       +--------------------+
                                                        |
                                                        v
                                               +--------------------+
                                               | Fact & Dimension   |
                                               | Tables             |
                                               | FACT_TRANSACTIONS  |
                                               | DIM_ACCOUNT        |
                                               | DIM_MERCHANT       |
                                               +--------------------+
                                                        |
                                                        v
                                               +--------------------+
                                               | Analytics / Views  |
                                               | V_ACCOUNT_SUMMARY  |
                                               +--------------------+

DESIGN DECISIONS

RAW Table First

All columns loaded as STRING to handle bad/malformed data

Preserves original source integrity

Staging Table

Columns cast to correct types

Minimal transformations; ready for incremental loads

Fact & Dimension Tables

Star schema design

Fact table stores measures (amounts, dates)

Dimension tables store descriptive attributes (accounts, merchants)

Business Rules

Removed zero/negative amounts

Standardized transaction types

Categorized merchants for analytics

Incremental Load

Snowflake STREAM tracks new rows

TASK schedules transformations automatically

Cost Controls

X-SMALL warehouse used (minimizes cost)

AUTO_SUSPEND = 60 seconds

TASK scheduled hourly (free-tier safe)

Temporary tables used for testing

Notes

Designed for free-tier Snowflake + AWS S3

Easily extended to production:

Use IAM roles instead of access keys

Ingest multiple CSV files

Implement error logging and alerting

Scripts & Queries

All SQL scripts used in this project:

01_create_file_format_and_stage.sql

02_create_raw_table.sql

03_load_raw.sql

04_create_staging_table.sql

05_create_fact_dim_tables.sql

06_apply_business_rules.sql

07_create_stream_and_task.sql