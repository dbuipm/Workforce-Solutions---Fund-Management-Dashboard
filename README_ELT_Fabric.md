# Workforce Solutions — Fund Management Dashboard (ELT Pipeline)

A production data engineering pipeline that extracts workforce development data from **Salesforce** via the REST API, loads it raw into **Microsoft Fabric Lakehouse**, transforms it using **Python and SQL**, and serves the cleaned data through a **Fabric Warehouse** to a **Power BI** dashboard.

This is the data engineering version of the Fund Management Dashboard. See [`README_LiveConnection.md`](./README_LiveConnection.md) for the direct Salesforce connector version.

---

## Table of Contents

- [Why ELT over Live Connection](#why-elt-over-live-connection)
- [Architecture Overview](#architecture-overview)
- [Pipeline Stages](#pipeline-stages)
- [Salesforce Extraction (Python)](#salesforce-extraction-python)
- [Fabric Lakehouse — Raw Layer](#fabric-lakehouse--raw-layer)
- [Transformation — Python & SQL](#transformation--python--sql)
- [Fabric Warehouse — Serving Layer](#fabric-warehouse--serving-layer)
- [Power BI Connection](#power-bi-connection)
- [Tables & Schema](#tables--schema)
- [Design Decisions](#design-decisions)
- [Reproducing the Pipeline](#reproducing-the-pipeline)
- [Skills Demonstrated](#skills-demonstrated)

---

## Why ELT over Live Connection

The live Salesforce connector (see other README) works well for real-time use, but has limitations at scale:

| Concern | Live Connection | This ELT Pipeline |
|---|---|---|
| Query performance | Constrained by Salesforce API limits | Runs on Fabric compute — no API limits during reporting |
| Data availability | Requires Salesforce org access for every user | Data lives in Fabric — accessible to any team member |
| Historical data | Current state only | Full history preserved in raw Lakehouse layer |
| Transformation logic | DAX only, runs at query time | Python + SQL, runs once at load time |
| Reproducibility | Tied to org credentials | Any engineer can connect and run the pipeline |
| Cross-system joins | Manual in Power BI | Pre-joined in warehouse, served as clean tables |

**The ELT approach also means the data connections persist permanently.** Any analyst, engineer, or future team member can connect directly to the Fabric Warehouse tables without needing Salesforce access or understanding the original object model.

---

## Architecture Overview

```
Salesforce REST API
        │
        │  Python (simple-salesforce)
        │  SOQL queries · Bulk API for large objects
        │  Incremental load by LastModifiedDate
        ▼
Fabric Lakehouse — Raw Layer
        │
        │  Delta tables, schema-on-read
        │  No transformations — exact API response preserved
        │  Partitioned by extraction date
        ▼
Dataflow Gen2 / Fabric Notebook
        │
        │  Python: deduplication, type casting, null handling
        │  SQL: relationship resolution, unified tables (Unified_SR_SB)
        │  Business logic: region mapping, CIP joins, fiscal year calc
        ▼
Fabric Warehouse — Serving Layer
        │
        │  Cleaned, typed, joined tables
        │  Permanent connections — always available
        │  SQL endpoint accessible to any tool
        ▼
Power BI (DirectQuery / Import)
        │
        │  Semantic model on top of Warehouse tables
        │  DAX measures for KPIs, projections, Top 15 rankings
        ▼
Fund Management Dashboard
```

---

## Pipeline Stages

### Stage 1 — Extract
Pull Salesforce objects via REST API using Python and `simple-salesforce`. Queries use SOQL with field-level selection. Large objects (Scholarship Request, Program Engagement) use the Bulk API. All responses land in the Lakehouse raw zone as Delta tables.

### Stage 2 — Load (Raw)
Raw data is written to the Fabric Lakehouse with no transformation — exact field names, raw values, nulls preserved. Each table is partitioned by `extraction_date` so historical snapshots are retained. This is the source of truth if anything goes wrong downstream.

### Stage 3 — Transform
Fabric Notebooks (Python + PySpark) and Dataflow Gen2 (SQL) clean the raw data:
- Cast data types (dates, decimals, booleans)
- Deduplicate records from incremental loads
- Resolve relationships between Salesforce objects
- Build the `Unified_SR_SB` table (the core design pattern — see below)
- Join ZIP codes to regions (East / North / West)
- Calculate fiscal year fields

### Stage 4 — Serve
Cleaned tables are written to the Fabric Warehouse. This layer has a stable SQL endpoint — any team member, tool, or future project can connect here without touching the pipeline.

### Stage 5 — Report
Power BI connects to the Warehouse via DirectQuery or scheduled Import. The semantic model adds DAX measures on top of the clean tables.

---

## Salesforce Extraction (Python)

### Setup

```bash
pip install simple-salesforce pandas pyarrow
```

### Authentication

```python
from simple_salesforce import Salesforce

sf = Salesforce(
    username=os.environ["SF_USERNAME"],
    password=os.environ["SF_PASSWORD"],
    security_token=os.environ["SF_TOKEN"],
    domain="login"  # or "test" for sandbox
)
```

> Credentials are stored as environment variables — never hardcoded. In Fabric, use Key Vault references or Notebook environment secrets.

### Standard object extraction (SOQL)

```python
def extract_object(sf, object_name: str, fields: list[str],
                   last_modified: str = None) -> pd.DataFrame:
    """
    Extract a Salesforce object with optional incremental filter.
    last_modified: ISO datetime string, e.g. '2026-01-01T00:00:00Z'
    """
    field_str = ", ".join(fields)
    query = f"SELECT {field_str} FROM {object_name}"

    if last_modified:
        query += f" WHERE LastModifiedDate >= {last_modified}"

    result = sf.query_all(query)
    records = result["records"]

    df = pd.DataFrame(records).drop(columns=["attributes"], errors="ignore")
    df["_extraction_date"] = pd.Timestamp.now().date()
    return df
```

### Bulk API for large objects

For objects with 100K+ rows (Scholarship Request, Program Engagement), use the Bulk API to avoid REST API governor limits:

```python
from simple_salesforce import SFBulkType

def bulk_extract(sf, object_name: str, fields: list[str]) -> pd.DataFrame:
    """Use Bulk API for high-volume objects."""
    field_str = ", ".join(fields)
    query = f"SELECT {field_str} FROM {object_name}"

    bulk = SFBulkType(object_name=object_name,
                      bulk_url=sf.bulk_url,
                      headers=sf.headers,
                      session=sf.session)

    results = bulk.query(query, lazy_operation=True)
    all_records = []
    for batch in results:
        all_records.extend(batch)

    df = pd.DataFrame(all_records)
    df["_extraction_date"] = pd.Timestamp.now().date()
    return df
```

### Objects extracted

| Salesforce Object | API Name | Method | Approx. Size |
|---|---|---|---|
| Scholarship Account | `Scholarship_Account__c` | REST | Small |
| Scholarship Budget | `Scholarship_Budget__c` | REST | Medium |
| Scholarship Request | `Scholarship_Request__c` | Bulk | Large |
| Funding Program | `Funding_Program__c` | REST | Small |
| Account (Customer) | `Account` | REST | Medium |
| Vendor Enrollment | `Vendor_Enrollment__c` | REST | Medium |
| Vendor Location | `Vendor_Location__c` | REST | Small |
| Program Cohort | `Program_Cohort__c` | REST | Small |
| Program Engagement | `Program_Engagement__c` | Bulk | Large |
| Program Location | `Program_Location__c` | REST | Small |
| CIP Code/Title | `CIP_Code_Title__c` | REST | Small |

### Writing to Fabric Lakehouse

```python
from deltalake import write_deltalake

def write_to_lakehouse(df: pd.DataFrame, table_name: str,
                       lakehouse_path: str, mode: str = "append"):
    """
    Write a DataFrame to the Fabric Lakehouse as a Delta table.
    mode: 'overwrite' for full refresh, 'append' for incremental
    """
    output_path = f"{lakehouse_path}/Tables/{table_name}"

    write_deltalake(
        output_path,
        df,
        mode=mode,
        partition_by=["_extraction_date"],
        schema_mode="merge"   # handles new columns added to SF objects
    )
    print(f"Written {len(df):,} rows to {output_path}")
```

### Incremental load pattern

```python
from delta import DeltaTable

def get_last_load_date(lakehouse_path: str, table_name: str) -> str | None:
    """Return the most recent extraction date from the Delta table."""
    try:
        dt = DeltaTable(f"{lakehouse_path}/Tables/{table_name}")
        df = dt.to_pandas(columns=["LastModifiedDate"])
        return df["LastModifiedDate"].max()
    except Exception:
        return None  # table doesn't exist yet — do full load

# Usage
last_date = get_last_load_date(LAKEHOUSE_PATH, "scholarship_request_raw")
df = extract_object(sf, "Scholarship_Request__c", SR_FIELDS, last_modified=last_date)
write_to_lakehouse(df, "scholarship_request_raw", LAKEHOUSE_PATH, mode="append")
```

---

## Fabric Lakehouse — Raw Layer

The raw layer stores one Delta table per Salesforce object. Nothing is modified from the API response — field names match Salesforce API names exactly, nulls are preserved, and every row has an `_extraction_date` partition column.

### Naming convention

```
lakehouse_raw/
└── Tables/
    ├── scholarship_account_raw/
    ├── scholarship_budget_raw/
    ├── scholarship_request_raw/
    ├── funding_program_raw/
    ├── account_raw/
    ├── vendor_enrollment_raw/
    ├── vendor_location_raw/
    ├── program_cohort_raw/
    ├── program_engagement_raw/
    ├── program_location_raw/
    └── cip_code_title_raw/
```

### Why raw-first matters

Storing the exact API response before any transformation means:
- If a transformation has a bug, you can re-run it against the same raw data without re-hitting the API
- Field renames or logic changes downstream don't require re-extraction
- Auditors or future engineers can see exactly what came out of Salesforce

---

## Transformation — Python & SQL

Transformations run in **Fabric Notebooks** (PySpark/Python) for complex logic and **Dataflow Gen2** (SQL) for straightforward cleaning and joins.

### Type casting and null handling

```python
from pyspark.sql import functions as F
from pyspark.sql.types import DecimalType, DateType, BooleanType

def clean_scholarship_budget(df_raw):
    return (
        df_raw
        # Cast financial fields
        .withColumn("Amount_Allocated_to_Customer_Accounts__c",
                    F.col("Amount_Allocated_to_Customer_Accounts__c")
                     .cast(DecimalType(18, 2)))
        .withColumn("Training_Scholarship_Budget__c",
                    F.col("Training_Scholarship_Budget__c")
                     .cast(DecimalType(18, 2)))
        # Cast dates
        .withColumn("Approval_Date__c",
                    F.to_date("Approval_Date__c"))
        # Standardize status values
        .withColumn("Status__c",
                    F.when(F.col("Status__c").isNull(), "Unknown")
                     .otherwise(F.col("Status__c")))
        # Deduplicate — keep latest record per Id
        .orderBy(F.col("LastModifiedDate").desc())
        .dropDuplicates(["Id"])
        # Drop extraction metadata before serving
        .drop("_extraction_date")
    )
```

### Building Unified_SR_SB

The most important transformation. This mirrors the DAX table from the live connection version but is built in PySpark, running once at load time rather than at every query:

```python
def build_unified_sr_sb(df_sr, df_sb):
    """
    Union Scholarship Requests with budget-only records (SBs with no SR).
    This prevents approved budgets from being dropped when joining through SR.
    """

    # SR fields to carry from the parent SB (via join)
    sb_cols = [
        "Id", "Status__c", "Amount_Allocated_to_Customer_Accounts__c",
        "Agreement_Decline_Reason__c", "AgreementsDate__c",
        "Career_Advisor__c", "Career_Office_Code__c",
        "Training_Scholarship_Budget__c", "Support_Scholarship_Budget__c",
        "Account__c", "Funding_Program__c"
    ]

    # Part 1: SR rows joined to their parent SB
    sr_with_sb = (
        df_sr
        .join(
            df_sb.select([F.col(c).alias(f"{c}_SB") for c in sb_cols]),
            df_sr["Scholarship_Budget__c"] == df_sb["Id_SB"],
            how="left"
        )
        .withColumn("Row_Type", F.lit("Request"))
        .withColumn("Unified_ID", F.concat(F.lit("SR-"), F.col("Id")))
    )

    # Part 2: SB-only rows — approved budgets with no linked SR
    sb_ids_with_sr = df_sr.select("Scholarship_Budget__c").distinct()

    sb_only = (
        df_sb
        .join(sb_ids_with_sr,
              df_sb["Id"] == sb_ids_with_sr["Scholarship_Budget__c"],
              how="left_anti")   # keep SBs NOT in SR
        .withColumn("Row_Type", F.lit("Budget"))
        .withColumn("Unified_ID", F.concat(F.lit("SB-"), F.col("Id")))
        # SR columns all null for budget-only rows
    )

    # Union both parts
    return sr_with_sb.unionByName(sb_only, allowMissingColumns=True)
```

### ZIP to region mapping (SQL)

```sql
-- Run in Dataflow Gen2 or Fabric Warehouse
CREATE OR REPLACE TABLE serving.account_with_region AS
SELECT
    a.*,
    COALESCE(r.Region, 'Unknown') AS Region
FROM
    cleaned.account a
LEFT JOIN
    raw.region_zip r
    ON a.BillingPostalCode = r.ZipCode;
```

### Fiscal year calculation

```python
def add_fiscal_year(df, date_col: str):
    """
    Workforce Solutions fiscal year runs Oct 1 – Sep 30.
    FY26 = Oct 2025 – Sep 2026.
    """
    return df.withColumn(
        "FiscalYear",
        F.when(F.month(date_col) >= 10,
               F.year(date_col) + 1)
         .otherwise(F.year(date_col))
    ).withColumn(
        "FiscalMonth",  # 1=Oct, 12=Sep
        F.when(F.month(date_col) >= 10,
               F.month(date_col) - 9)
         .otherwise(F.month(date_col) + 3)
    )
```

---

## Fabric Warehouse — Serving Layer

The Warehouse is the permanent serving layer. Tables here are clean, typed, joined, and always available via SQL endpoint — no pipeline knowledge required to use them.

### Schema

```
warehouse_serving/
├── schema: raw_passthrough/     -- exact lakehouse tables, SQL-accessible
└── schema: serving/             -- cleaned and joined tables for reporting
    ├── unified_sr_sb            -- central fact table (SR + orphaned SBs)
    ├── scholarship_budget       -- one row per customer per program
    ├── scholarship_request      -- individual spend requests
    ├── funding_program          -- program-level budgets and MIP codes
    ├── account_with_region      -- customers with region assignment
    ├── vendor_enrollment        -- vendor-cohort-CIP mappings
    ├── vendor_location          -- vendor addresses (for map visual)
    ├── program_cohort           -- training cohort metadata
    ├── program_engagement       -- customer-to-program enrollments
    ├── cip_code_title           -- occupation classification codes
    └── fiscal_calendar          -- FY date spine (Oct–Sep)
```

### Connecting to the Warehouse

Any tool that supports SQL Server / TDS protocol can connect:

```
Server:   <workspace>.datawarehouse.fabric.microsoft.com
Database: warehouse_serving
Auth:     Azure AD (Microsoft account)
```

**Python (pyodbc):**
```python
import pyodbc

conn = pyodbc.connect(
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=<workspace>.datawarehouse.fabric.microsoft.com;"
    "Database=warehouse_serving;"
    "Authentication=ActiveDirectoryInteractive;"
)

df = pd.read_sql("SELECT * FROM serving.unified_sr_sb WHERE Status__c_SB = 'Approved'", conn)
```

**Power BI:** Connect via Get Data → Microsoft Fabric → Warehouse, select `warehouse_serving`.

---

## Power BI Connection

Power BI connects to the Fabric Warehouse, not directly to Salesforce. This means:
- Any report creator with Fabric access can build dashboards without Salesforce credentials
- Multiple reports can share the same cleaned data without duplicating transformation logic
- The semantic model adds DAX measures on top of the warehouse tables

### Import vs DirectQuery

| Mode | Best for | Trade-off |
|---|---|---|
| Import | Fast dashboard performance | Data is a snapshot — refreshes on schedule |
| DirectQuery | Always-current data | Slower queries — hits Warehouse on every interaction |

For this dashboard, **Import mode with scheduled refresh** is recommended. Fund data doesn't change by the second — a daily or hourly refresh is sufficient and delivers much faster report performance.

---

## Tables & Schema

### `serving.unified_sr_sb` — Central fact table

| Column | Type | Source | Description |
|---|---|---|---|
| `Unified_ID` | VARCHAR | Derived | `"SR-" + Id` or `"SB-" + Id` |
| `Row_Type` | VARCHAR | Derived | `"Request"` or `"Budget"` |
| `Id` | VARCHAR | SR | Scholarship Request ID |
| `Id_SB` | VARCHAR | SB | Scholarship Budget ID |
| `Status__c_SB` | VARCHAR | SB | Budget status (filter to `"Approved"`) |
| `Amount_Allocated_to_Customer_Accounts__c_SB` | DECIMAL(18,2) | SB | Allocated amount per customer |
| `Program_Type__c` | VARCHAR | SR | Training / Support / Gift Card |
| `Request_Status__c` | VARCHAR | SR | Authorized / Pending / Denied |
| `Amount__c` | DECIMAL(18,2) | SR | Request amount |
| `Account__c_SB` | VARCHAR | SB | Customer account ID (FK → account) |
| `Funding_Program__c_SB` | VARCHAR | SB | Funding program ID (FK → funding_program) |
| `FiscalYear` | INT | Derived | e.g. `2026` |
| `FiscalMonth` | INT | Derived | 1=Oct … 12=Sep |

### `serving.funding_program`

| Column | Type | Description |
|---|---|---|
| `Id` | VARCHAR | Primary key |
| `Name` | VARCHAR | Program name (e.g. "WIOA: Adult Training - 2026") |
| `Accounting_Code__c` | VARCHAR | MIP fund code |
| `MIP_Original_Amount` | DECIMAL(18,2) | Original budget from MIP |
| `Available_Amount__c` | DECIMAL(18,2) | Available per Salesforce |
| `Committed_Amount__c` | DECIMAL(18,2) | Committed amount |
| `Consumed_Amount__c` | DECIMAL(18,2) | Consumed / expensed |
| `StartDate` | DATE | Fiscal period start |
| `EndDate` | DATE | Fiscal period end |

---

The semantic model, DAX measures, and dashboard design are identical to the live connection version — see README_LiveConnection.md for full documentation.



## Design Decisions

### ELT not ETL
Data lands raw first, then gets transformed inside Fabric. This means the raw layer is always available as a recovery point, and transformation logic can be changed and re-run without re-hitting the Salesforce API.

### Incremental load by LastModifiedDate
Rather than pulling all records every run, the pipeline filters by `LastModifiedDate >= last_extraction`. This reduces API call volume and runtime significantly for large objects like Scholarship Request.

### Bulk API for high-volume objects
Salesforce REST API has governor limits (concurrent API calls, daily limits). Objects with 50K+ rows use the Bulk API, which processes asynchronously and returns results in batches — no governor pressure.

### Schema merge on Delta write
Delta Lake's `schema_mode="merge"` means new fields added to Salesforce objects are automatically picked up on the next run without breaking the pipeline. New columns appear in the raw table and propagate through transformations as nulls until explicitly handled.

### Unified_SR_SB built in PySpark, not DAX
In the live connection version, this table is built as a calculated DAX table at query time. In this pipeline it's built once in PySpark and stored in the warehouse — significantly faster for large datasets and available to any downstream tool, not just Power BI.

### Permanent SQL endpoint
The Fabric Warehouse SQL endpoint is always on. Any analyst with access can query `serving.unified_sr_sb` directly in SQL, connect from Excel, or build a new Power BI report — without going through the pipeline or touching Salesforce.

---

## Reproducing the Pipeline

### Prerequisites

- Microsoft Fabric workspace (with Lakehouse and Warehouse created)
- Salesforce org credentials (username, password, security token)
- Python 3.10+

### Install dependencies

```bash
pip install simple-salesforce pandas pyarrow deltalake pyspark
```

### Environment variables

```bash
export SF_USERNAME="your@email.com"
export SF_PASSWORD="yourpassword"
export SF_TOKEN="yoursecuritytoken"
export LAKEHOUSE_PATH="abfss://workspace@onelake.dfs.fabric.microsoft.com/lakehouse_raw.Lakehouse"
```

### Run order

```bash
# 1. Extract all Salesforce objects to raw Lakehouse
python extract/run_extraction.py

# 2. Clean and transform (run in Fabric Notebook or locally with PySpark)
python transform/clean_all.py

# 3. Build Unified_SR_SB
python transform/build_unified_sr_sb.py

# 4. Write serving tables to Warehouse
python serve/write_to_warehouse.py

# 5. Refresh Power BI dataset (via API or manually)
python serve/trigger_pbi_refresh.py
```

### Scheduling

In production, this pipeline runs as a **Fabric Data Pipeline** on a daily schedule. Each stage is a separate activity in the pipeline, with dependency chains so transformation only runs after a successful extraction.

---

## Skills Demonstrated

**Data Engineering**
- Salesforce REST and Bulk API integration using `simple-salesforce`
- Incremental load pattern with `LastModifiedDate` watermarking
- Delta Lake writes with schema merge and date partitioning
- ELT architecture — raw preservation before transformation

**Python / PySpark**
- PySpark DataFrame transformations (type casting, deduplication, null handling)
- Complex union patterns to handle optional relationships (`left_anti` join for orphaned records)
- Parameterized extraction functions for reusability across all Salesforce objects

**Microsoft Fabric**
- Lakehouse raw zone design (Delta tables, partitioning strategy)
- Dataflow Gen2 and Notebook orchestration
- Fabric Warehouse SQL endpoint as a permanent serving layer
- Power BI DirectQuery and Import mode trade-offs

**Data Modeling**
- Designing `Unified_SR_SB` as a PySpark transformation rather than a DAX table — same business logic, scalable execution
- Fiscal year calendar generation (Oct–Sep, non-standard)
- ZIP-to-region lookup join for geographic segmentation

**Software Engineering Practices**
- Credentials managed via environment variables and Key Vault references
- Schema-on-read for raw layer, schema-on-write for serving layer
- Recovery design: raw layer as source of truth, transformations re-runnable at any time
