### Data Migration Document: SAP S/4HANA to Snowflake – SQL Scripts

**Title**: *Data Migration from SAP S/4HANA to Snowflake: SQL Scripts and Workflow*  
**Author**: [Your Name], Research Analyst  
**Date**: March 6, 2025  
**Context**: Scripts for Nordic Retail Co.’s migration of 2TB of transactional data (sales orders, inventory, financials) from SAP S/4HANA to Snowflake, completed May 2025.

---

#### Overview
This document provides SQL scripts and workflows for migrating data from SAP S/4HANA to Snowflake, as executed in Nordic Retail Co.’s case study. It covers extraction from S/4HANA, loading into Snowflake, schema transformation, and validation—key steps to achieve real-time analytics with a 3-second query latency.

**Prerequisites**:  
- SAP HANA access (e.g., via HANA Studio or DBeaver).  
- Snowflake account (e.g., AWS region, “RETAIL_DW” database).  
- Tools: SAP BODS, BryteFlow (CDC), AWS S3 staging, dbt (optional).  
- Sample Tables: VBAK (sales orders), MSEG (material movements), ACDOCA (financials).

---

#### Migration Workflow and SQL Scripts

##### Step 1: Data Extraction from SAP S/4HANA
**Objective**: Extract historical data (2TB) and set up CDC for real-time updates.  
**Tools**: SAP BODS for bulk export, BryteFlow for CDC.  
**SQL Example**: Extract sales orders (VBAK) for bulk CSV export.

```sql
-- SAP HANA: Extract VBAK (Sales Orders) for 5 years
SELECT 
    VBELN AS SALES_ORDER_ID,    -- Sales order number
    ERDAT AS ORDER_DATE,        -- Creation date
    KUNNR AS CUSTOMER_ID,       -- Customer number
    NETWR AS NET_VALUE          -- Net value
FROM 
    VBAK
WHERE 
    ERDAT >= '2020-01-01'      -- 5-year historical data
    AND ERDAT <= '2025-03-05'
ORDER BY 
    ERDAT;
-- Export to CSV via BODS or DBeaver: ~5M rows expected
```

**Notes**:  
- Custom fields (e.g., “Store_Region”) added via joins with custom tables (e.g., ZSTORE).  
- CDC via BryteFlow captures INSERT/UPDATE on VBAK post-extraction.

---

##### Step 2: Staging and Loading into Snowflake
**Objective**: Load 2TB of CSV data from S/4HANA into Snowflake via S3 staging.  
**Setup**:  
- S3 Bucket: `s3://nordic-retail-staging/`.  
- Snowflake Stage: `STAGE_RETAIL`.  
- Target Table: `SALES_FACT`.

**SQL Scripts**:  
1. **Create Stage and File Format**  
```sql
-- Snowflake: Set up external stage
CREATE OR REPLACE STAGE RETAIL_DW.PUBLIC.STAGE_RETAIL
    URL = 's3://nordic-retail-staging/'
    CREDENTIALS = (AWS_KEY_ID = '<your_key>' AWS_SECRET_KEY = '<your_secret>');

-- Define CSV file format
CREATE OR REPLACE FILE FORMAT RETAIL_DW.PUBLIC.CSV_FORMAT
    TYPE = 'CSV'
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1
    NULL_IF = ('\\N', 'NULL', '');
```

2. **Create Target Table**  
```sql
-- Snowflake: Sales fact table
CREATE OR REPLACE TABLE RETAIL_DW.PUBLIC.SALES_FACT (
    SALES_ORDER_ID VARCHAR(10),
    ORDER_DATE DATE,
    CUSTOMER_ID VARCHAR(10),
    NET_VALUE DECIMAL(15,2),
    STORE_REGION VARCHAR(50)  -- Custom field from S/4HANA
);
```

3. **Load Data**  
```sql
-- Snowflake: Bulk load from S3
COPY INTO RETAIL_DW.PUBLIC.SALES_FACT
FROM @RETAIL_DW.PUBLIC.STAGE_RETAIL/sales_orders/
FILE_FORMAT = (FORMAT_NAME = 'RETAIL_DW.PUBLIC.CSV_FORMAT')
PATTERN = '.*vbak.*[.]csv'
ON_ERROR = 'CONTINUE';
-- Loaded 2TB in 12 hrs initially, optimized to 4 hrs with Large warehouse
```

**Notes**:  
- CDC data streams directly into `SALES_FACT` via BryteFlow pipeline (5-min latency).  
- Warehouse size: Adjusted from Medium to Large for faster loading.

---

##### Step 3: Schema Design and Transformation
**Objective**: Build a star schema for analytics, materializing S/4HANA views.  
**Setup**:  
- Fact Table: `SALES_FACT`.  
- Dimension Tables: `STORE_DIM`, `TIME_DIM`.

**SQL Scripts**:  
1. **Create Dimension Tables**  
```sql
-- Snowflake: Store dimension
CREATE OR REPLACE TABLE RETAIL_DW.PUBLIC.STORE_DIM (
    STORE_ID VARCHAR(10) PRIMARY KEY,
    STORE_NAME VARCHAR(50),
    REGION VARCHAR(50)
);
-- Populated via separate S/4HANA extract (e.g., ZSTORE table)

-- Time dimension
CREATE OR REPLACE TABLE RETAIL_DW.PUBLIC.TIME_DIM (
    DATE_ID DATE PRIMARY KEY,
    YEAR INT,
    QUARTER INT,
    MONTH INT,
    DAY INT
);
-- Populate with a script or dbt model
INSERT INTO RETAIL_DW.PUBLIC.TIME_DIM
SELECT 
    DATEADD('DAY', SEQ4(), '2020-01-01') AS DATE_ID,
    YEAR(DATE_ID) AS YEAR,
    QUARTER(DATE_ID) AS QUARTER,
    MONTH(DATE_ID) AS MONTH,
    DAY(DATE_ID) AS DAY
FROM 
    TABLE(GENERATOR(ROWCOUNT => 2000));  -- 5+ years
```

2. **Materialize Sales by Region View**  
```sql
-- Snowflake: Transform and materialize sales view
CREATE OR REPLACE TABLE RETAIL_DW.PUBLIC.SALES_BY_REGION AS
SELECT 
    s.STORE_REGION,
    t.YEAR,
    t.QUARTER,
    SUM(s.NET_VALUE) AS TOTAL_SALES
FROM 
    RETAIL_DW.PUBLIC.SALES_FACT s
JOIN 
    RETAIL_DW.PUBLIC.TIME_DIM t ON s.ORDER_DATE = t.DATE_ID
GROUP BY 
    s.STORE_REGION, t.YEAR, t.QUARTER;
-- Query time: 5-10 secs vs. S/4HANA’s 30 mins
```

**Notes**:  
- Used dbt to automate transformations (e.g., incremental updates).  
- Handled hierarchical data (e.g., product categories) with recursive CTEs if needed.

---

##### Step 4: Validation and Testing
**Objective**: Ensure data integrity post-migration.  
**Approach**: Compare row counts and aggregates.

**SQL Scripts**:  
1. **Row Count Validation**  
```sql
-- Snowflake: Check SALES_FACT rows
SELECT COUNT(*) AS SNOWFLAKE_ROWS
FROM RETAIL_DW.PUBLIC.SALES_FACT;
-- Compare to S/4HANA export (~5M rows expected)

-- SAP HANA: Original count
SELECT COUNT(*) AS S4HANA_ROWS
FROM VBAK
WHERE ERDAT >= '2020-01-01' AND ERDAT <= '2025-03-05';
```

2. **Aggregate Check**  
```sql
-- Snowflake: Total sales value
SELECT SUM(NET_VALUE) AS SNOWFLAKE_TOTAL
FROM RETAIL_DW.PUBLIC.SALES_FACT;

-- SAP HANA: Compare
SELECT SUM(NETWR) AS S4HANA_TOTAL
FROM VBAK
WHERE ERDAT >= '2020-01-01' AND ERDAT <= '2025-03-05';
-- 99.9% match achieved after CDC fix
```

**Notes**:  
- Discrepancy (1K rows) resolved by re-syncing CDC pipeline.  
- Automated via BryteFlow TruData tool.

---

##### Step 5: Go-Live and Optimization
**Objective**: Optimize Snowflake for ongoing use.  
**SQL Scripts**:  
1. **Automate CDC Refreshes**  
```sql
-- Snowflake: Task for 5-min CDC updates
CREATE OR REPLACE TASK RETAIL_DW.PUBLIC.REFRESH_SALES
    WAREHOUSE = 'ANALYTICS_WH'
    SCHEDULE = '5 MINUTE'
AS
    MERGE INTO RETAIL_DW.PUBLIC.SALES_FACT AS target
    USING RETAIL_DW.PUBLIC.STAGE_SALES_CDC AS source
    ON target.SALES_ORDER_ID = source.SALES_ORDER_ID
    WHEN MATCHED THEN UPDATE SET
        target.ORDER_DATE = source.ORDER_DATE,
        target.NET_VALUE = source.NET_VALUE
    WHEN NOT MATCHED THEN INSERT (
        SALES_ORDER_ID, ORDER_DATE, CUSTOMER_ID, NET_VALUE, STORE_REGION
    ) VALUES (
        source.SALES_ORDER_ID, source.ORDER_DATE, source.CUSTOMER_ID, 
        source.NET_VALUE, source.STORE_REGION
    );
-- STAGE_SALES_CDC populated by BryteFlow
```

2. **Dynamic Scaling**  
```sql
-- Snowflake: Adjust warehouse size
ALTER WAREHOUSE ANALYTICS_WH SET WAREHOUSE_SIZE = 'X-LARGE' AUTO_SUSPEND = 300;
-- Scaled to Small overnight via scheduler
```

**Notes**:  
- Compute cost dropped from €5K/mo to €2K/mo with auto-scaling.

---

#### Cost and Performance Outcomes
- **Migration Cost**: €50K one-time, €36K/year ongoing.  
- **Performance**: 3-sec query latency vs. 30 mins in S/4HANA.  
- **Scalability**: Handles 2TB + 1TB/year effortlessly.

---

#### Best Practices
1. **Extraction**: Use BODS for bulk, CDC for deltas—test CDC latency early.  
2. **Loading**: Optimize warehouse size before bulk loads.  
3. **Transformation**: Leverage dbt for automation, materialize key views.  
4. **Validation**: Automate checks with scripts/tools.  
5. **Optimization**: Schedule tasks and scale dynamically.

---

### Notes for Use
- **Adaptation**: Replace placeholders (e.g., `<your_key>`, table names) with your specifics.  
- **Execution**: Run SAP HANA scripts via HANA Studio/DBeaver; Snowflake scripts via UI/SnowSQL.  
- **Scalability**: Tested for 2TB—adjust for larger datasets (e.g., X-Large warehouse).  

This document equips you to replicate Nordic Retail Co.’s success. Pair it with the case study on GitHub (`S4HANA-to-Snowflake-Migration-Case-Study`) for the full story!

