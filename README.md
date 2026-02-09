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

```markdown
## Design Decisions

### RAW Table
- All columns loaded as STRING to handle bad/malformed data
- Preserves original source integrity

### Staging Table
- Columns cast to correct types
- Minimal transformations; ready for incremental loads

### Fact & Dimension Tables
- Star schema design
- Fact table stores measures (amounts, dates)
- Dimension tables store descriptive attributes (accounts, merchants)

### Business Rules
- Removed zero/negative amounts
- Standardized transaction types
- Categorized merchants for analytics

### Incremental Load
- Snowflake STREAM tracks new rows
- TASK schedules transformations automatically