# Finance Data Pipeline Demo

## Overview
This project demonstrates an end-to-end **AWS S3 ➞ Snowflake ETL pipeline** for financial transaction data.  
It covers:

- Raw data ingestion  
- Staging table creation & cleaning  
- Fact & dimension table creation  
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
```

---

## Design Decisions

### RAW Table
- All columns loaded as **STRING** to handle bad/malformed data  
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
- Snowflake **STREAM** tracks new rows  
- **TASK** schedules transformations automatically

---

## Cost Controls

- **X-SMALL warehouse** used (minimizes cost)  
- **AUTO_SUSPEND = 60 seconds**  
- TASK scheduled hourly (free-tier safe)  
- Temporary tables used for testing

---

## Notes

- Designed for **free-tier Snowflake + AWS S3**  
- Easily extended to production:
  - Use IAM roles instead of access keys  
  - Ingest multiple CSV files  
  - Implement error logging and alerting

---

## Scripts & Queries

<details>
<summary><strong>01_create_file_format_and_stage.sql</strong></summary>

```sql
-- File Format
CREATE OR REPLACE FILE FORMAT FINANCE_CSV_FORMAT
TYPE = 'CSV'
FIELD_DELIMITER = ','
SKIP_HEADER = 1
FIELD_OPTIONALLY_ENCLOSED_BY = '"'
NULL_IF = ('NULL', 'null', '');

-- Stage pointing to S3
CREATE OR REPLACE STAGE FINANCE_RAW_STAGE
URL = 's3://your-bucket-name/raw/'
CREDENTIALS = (
  AWS_KEY_ID = '<ACCESS_KEY>'
  AWS_SECRET_KEY = '<SECRET_KEY>'
)
FILE_FORMAT = FINANCE_DB.PUBLIC.FINANCE_CSV_FORMAT;
```

</details>

<details>
<summary><strong>02_create_raw_table.sql</strong></summary>

```sql
-- RAW Table (12 STRING columns)
CREATE OR REPLACE TABLE FINANCE_DB.PUBLIC.RAW_TRANSACTIONS (
    COL1 STRING, COL2 STRING, COL3 STRING, COL4 STRING,
    COL5 STRING, COL6 STRING, COL7 STRING, COL8 STRING,
    COL9 STRING, COL10 STRING, COL11 STRING, COL12 STRING
);
```

</details>

<details>
<summary><strong>03_load_raw.sql</strong></summary>

```sql
COPY INTO FINANCE_DB.PUBLIC.RAW_TRANSACTIONS
FROM @FINANCE_RAW_STAGE
FILE_FORMAT = (FORMAT_NAME = FINANCE_DB.PUBLIC.FINANCE_CSV_FORMAT)
ON_ERROR = 'CONTINUE';
```

</details>

<details>
<summary><strong>04_create_staging_table.sql</strong></summary>

```sql
CREATE OR REPLACE TABLE FINANCE_DB.PUBLIC.STG_TRANSACTIONS AS
SELECT
    TRY_TO_DATE(COL1, 'YYYY-MM-DD') AS TRANSACTION_DATE,
    COL2 AS ACCOUNT_ID,
    TRY_TO_NUMBER(COL3) AS AMOUNT,
    COL4 AS TRANSACTION_TYPE,
    COL5 AS DESCRIPTION,
    COL6 AS CATEGORY,
    COL7 AS MERCHANT,
    COL8 AS LOCATION,
    COL9 AS CURRENCY,
    COL10 AS STATUS,
    COL11 AS CHANNEL,
    COL12 AS REMARKS
FROM FINANCE_DB.PUBLIC.RAW_TRANSACTIONS;
```

</details>

<details>
<summary><strong>05_create_fact_dim_tables.sql</strong></summary>

```sql
-- Accounts dimension
CREATE OR REPLACE TABLE FINANCE_DB.PUBLIC.DIM_ACCOUNT AS
SELECT DISTINCT ACCOUNT_ID
FROM FINANCE_DB.PUBLIC.STG_TRANSACTIONS;

-- Merchants dimension
CREATE OR REPLACE TABLE FINANCE_DB.PUBLIC.DIM_MERCHANT AS
SELECT DISTINCT MERCHANT, LOCATION, CURRENCY
FROM FINANCE_DB.PUBLIC.STG_TRANSACTIONS;

-- Fact table
CREATE OR REPLACE TABLE FINANCE_DB.PUBLIC.FACT_TRANSACTIONS AS
SELECT
    TRANSACTION_DATE,
    ACCOUNT_ID,
    AMOUNT,
    TRANSACTION_TYPE,
    CATEGORY,
    MERCHANT,
    CURRENCY,
    STATUS,
    CHANNEL
FROM FINANCE_DB.PUBLIC.STG_TRANSACTIONS;
```

</details>

<details>
<summary><strong>06_apply_business_rules.sql</strong></summary>

```sql
-- Remove negative or zero amounts
DELETE FROM FACT_TRANSACTIONS WHERE AMOUNT <= 0;

-- Standardize transaction types
UPDATE FACT_TRANSACTIONS
SET TRANSACTION_TYPE = UPPER(TRANSACTION_TYPE);

-- Categorize merchants example
UPDATE FACT_TRANSACTIONS
SET CATEGORY = 'ONLINE_SHOPPING'
WHERE MERCHANT ILIKE '%Amazon%' OR MERCHANT ILIKE '%Takealot%';
```

</details>

<details>
<summary><strong>07_create_stream_and_task.sql</strong></summary>

```sql
-- Stream to capture new rows in RAW table
CREATE OR REPLACE STREAM FINANCE_DB.PUBLIC.RAW_TRANSACTIONS_STREAM
ON TABLE FINANCE_DB.PUBLIC.RAW_TRANSACTIONS
APPEND_ONLY = TRUE;

-- Task to move new rows to staging automatically
CREATE OR REPLACE TASK FINANCE_DB.PUBLIC.LOAD_STG_TASK
WAREHOUSE = DEMO_WH
SCHEDULE = 'USING CRON 0 * * * * UTC'
AS
INSERT INTO FINANCE_DB.PUBLIC.STG_TRANSACTIONS
SELECT
    TRY_TO_DATE(COL1, 'YYYY-MM-DD') AS TRANSACTION_DATE,
    COL2 AS ACCOUNT_ID,
    TRY_TO_NUMBER(COL3) AS AMOUNT,
    COL4 AS TRANSACTION_TYPE,
    COL5 AS DESCRIPTION,
    COL6 AS CATEGORY,
    COL7 AS MERCHANT,
    COL8 AS LOCATION,
    COL9 AS CURRENCY,
    COL10 AS STATUS,
    COL11 AS CHANNEL,
    COL12 AS REMARKS
FROM FINANCE_DB.PUBLIC.RAW_TRANSACTIONS_STREAM;

-- Resume the task to start scheduling
ALTER TASK FINANCE_DB.PUBLIC.LOAD_STG_TASK RESUME;
```

</details>

---

## How to Use

1. Create all SQL files inside a `/sql` folder  
2. Configure AWS S3 and Snowflake environment  
3. Run scripts in numerical order to deploy pipeline  
4. Validate data at each stage  
5. Monitor costs in Snowflake UI

---

## Author

Created by **Zaya267** — AWS + Snowflake finance data pipeline demo for portfolio/interview purposes.

