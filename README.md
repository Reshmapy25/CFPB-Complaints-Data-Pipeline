# CFPB Consumer Complaints Analytics Platform

<img width="1034" height="589" alt="CFPB dashboard" src="https://github.com/user-attachments/assets/f17dbc4b-cb99-4515-b0e5-6a1c70230d89" />


An end-to-end data pipeline and dashboard built on **2,023,066 consumer financial complaints** from the CFPB Consumer Complaint Database (March 2015 to July 2024). Raw data moves through a Databricks medallion architecture (Bronze, Silver, Gold), is loaded into Snowflake, and is reported in Power BI so operations teams can see where complaints are concentrated.
---

## Business problem

Detect complaint hotspots early, so operations teams can prioritise high-volume problem areas and investigate them before they grow.

**Primary KPI:** distinct complaint count.

## Key findings

| Finding | Result |
|---|---|
| Share of complaints about credit reporting | **59.6%** (1,205,275 of 2,023,066) |
| Share held by three credit bureaus (Equifax, TransUnion, Experian) | **53.0%** (1,072,190 complaints) |
| Largest single company | Equifax, 372,385 complaints |
| Complaints that received a late company response | **1.3%** (26,304) |
| Product groups | 5: Credit Reporting, Debt Collection, Loans, Credit Card Services, Bank Accounts and Services |

## Architecture

```text
CFPB complaints CSV (2,023,066 rows)
        |
        v
Databricks, PySpark, Delta Lake (medallion architecture)
  Bronze: raw data kept unchanged, plus ingestion metadata
  Silver: cleaned, standardised, validated
  Gold:   KPI summary tables
        |
        v
Snowflake (CFPB_DB.GOLD): cleaned Gold tables
        |
        v
Power BI: dashboard
```

| Layer | What it does |
|---|---|
| **Bronze** | Loads the raw CSV as a Delta table with `_ingestion_timestamp`, `_source_file` and `_source_system` columns. Only column names change (spaces to underscores), because Delta rejects spaces. |
| **Silver** | Renames columns to lowercase snake case, converts the text `"null"` to real nulls, trims text, converts `date_received` from text to a date, fills missing `sub_product` and `sub_issue` with "Not specified" and missing `state` with "Unknown", and confirms every `complaint_id` is unique. No rows are dropped. |
| **Gold** | Three summary tables: complaints by month and product, complaints by company, and complaints by company and product. |
| **Snowflake** | Gold tables loaded and rebuilt as `_CLEAN` tables with correct names and data types. |
| **Power BI** | Imports the clean Snowflake tables and reports the KPI, trends and top companies. |

## Data quality and validation

The total of **2,023,066 distinct complaints** was checked at every stage: Bronze, Silver, each Gold table, Snowflake and Power BI. If any step lost or double counted a record, the totals would not match.

| Check | Result |
|---|---|
| Rows in Bronze and Silver | 2,023,066 |
| Distinct complaint IDs | 2,023,066 (no duplicates) |
| Rows with no `date_received` after conversion | 0 |
| Missing `sub_issue` | 230,559 (11.4%) |
| Missing `sub_product` | 52,206 (2.6%) |
| Missing `state` | 7,344 (0.4%) |

## Challenges and fixes

| Problem | Fix |
|---|---|
| The source file marks missing values with the text `null`, so null checks missed them | Trimmed the text and converted `"null"` to real nulls in Silver |
| The Databricks serverless Snowflake connector blocked its write options | Exported the small Gold tables and loaded them through the Snowflake web interface |
| Snowflake stored the loaded columns as text (`C1`, `C2`, `C3`) and the header line as a data row | Rebuilt each table with `CREATE TABLE ... AS SELECT`, using `TRY_TO_DATE` and `TRY_CAST` to set types and the `WHERE` clause to remove the header row |
| Power BI showed errors in the month column | Traced it to the text storage in Snowflake and fixed the data at the source |

## Tech stack

- **Processing:** Databricks, PySpark, Delta Lake
- **Warehouse:** Snowflake (SQL)
- **Reporting:** Power BI
- **Language:** Python, SQL

## Repository contents

```text
notebooks/
  01_cfpb_complaints_bronze
  02_cfpb_complaints_silver
  03_cfpb_complaints_gold
sql/
  snowflake setup and clean-table scripts
dashboard/
  CFPB_Complaints_Dashboard.pbix
  dashboard_preview.png
```

## Data source

[CFPB Consumer Complaint Database](https://www.consumerfinance.gov/data-research/consumer-complaints/), a public dataset from the U.S. Consumer Financial Protection Bureau.

## Author

**Reshma Babu**
[LinkedIn](https://linkedin.com/in/reshma-babu-vaishak) | [GitHub](https://github.com/Reshmapy25)
