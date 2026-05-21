# Workforce Solutions — Fund Management Dashboard (Power BI)

A production-grade Power BI dashboard built to track and analyze workforce development funding across federal and state programs. This is used operationally by **Workforce Solutions Gulf Coast** — one of the largest workforce boards in the US — to monitor how scholarship funds flow from program budgets down to individual customer training accounts.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Dashboard Pages](#dashboard-pages)
- [Data Sources](#data-sources)
- [Data Model](#data-model)
- [The Unified\_SR\_SB Table — Core Design Decision](#the-unified_sr_sb-table--core-design-decision)
- [DAX Measures — Key Patterns & Design Decisions](#dax-measures--key-patterns--design-decisions)
- [UX & Navigation Design](#ux--navigation-design)
- [Full Measure List](#full-measure-list)
- [Skills Demonstrated](#skills-demonstrated)

---

## Project Overview

Workforce Solutions administers millions of dollars in annual funding from programs like **WIOA, TANF, SNAP E&T, NDW, Child Care Quality, Skills Development Fund**, and others. Customers (job seekers) apply for scholarship budgets to attend training programs at approved vendors, and financial aid staff need a clear, real-time view of where every dollar is going.

**The dashboard answers three core questions:**

- **How much?** — What's been allocated, committed, encumbered, and expended per program?
- **How many?** — How many customers, enrollments, and training vs. support requests exist?
- **Where is it going?** — Which vendors, occupations (CIPs), service providers, and regions are receiving the funds?

---

## Dashboard Pages

### Funds (Home)
The entry point. Displays all funding programs as stacked bar charts with four color-coded segments — Available to System (green), Committed (light blue), Encumbered (dark blue), and Paid (navy). Programs are grouped into three tabs: **WIOA**, **TANF/SNAP/NDW**, and **Others**. A tooltip on hover shows original budget, available amount, and % obligated. Clicking any bar drills through to the Fund Details page.

### Fund Details
A deep-dive into one selected funding program. Shows:
- A horizontal bar chart of Total Committed / Encumbered / Paid broken out by **region** (East, North, West) and service provider
- Two pie charts: Total Allocated vs. Available to System, and Accounts with Balance vs. Accounts with Zero Balance
- A drill-down matrix table showing region → service provider → individual account counts, committed, encumbered, paid, and % allocated

### Occupations / CIP
Analyzes how funds are being used by **occupation type** (using CIP codes — Classification of Instructional Programs). Three sub-views switchable via buttons:
- **Occupation** — Top 15 CIP families by allocated amount, encumbered, and total expenditures; enrollment counts; Top 15 vs. All Others donut comparison
- **Provider** — Same metrics broken down by vendor name, plus an enrollment efficiency ratio (enrollments per dollar)
- **Program** — Breakdown by training cohort name

### Expenditures
Shows how spending is distributed across service providers and regions:
- **Service Provider view** — Usage % by provider (BakerRipley, SERCO, EDSI), allocation and customer distribution pie charts, and a detail table with training vs. support request counts
- **Region view** — Same metrics sliced by East / North / West
- **Funding Program view** — Full budget performance table: available financial aid budget, unallocated amount, % available, % allocated, % expensed, plus a second table with customer counts, training/support scholarship counts, training and support budgets, and expenses

### Projection
Forward-looking view for **WIOA and TANF/SNAP/NDW programs**. For each fund:
- Estimated exhaustion date (based on current daily allocation run rate)
- Projected allocated per day and per month
- Projected total allocated through the fiscal year end vs. actual allocated to date
- Percentage obligated
- A **Gantt-style heatmap** showing Oct–Sep months, color-coded green (on track) or red (at risk / already exhausted)

Tooltip on any row shows: fund name, start/end date, projected allocated per day, estimated exhaustion date, and days remaining.

### Projection — Others
Same projection logic applied to **Other funds** (Child Care Quality CCQ/CQF, Local Workforce Initiatives, Skills Development Fund-NRG Energy, WIOA Wagner Peyser, TEA Regional Convener). These use MIP expenditure data rather than FAMS allocation data.

### Vendor Enrollment
Shows enrollment counts (number of customer accounts) by vendor, with drill-down to service provider sub-locations and individual scholarship account IDs. Includes:
- Top 15 Vendors by Enrollment bar chart
- Service Provider Enrollment summary (BakerRipley, SERCO, EDSI)
- Drill-through to Vendor Maps

### Vendor Maps
An Azure Maps visual plotting all vendor locations as bubbles sized by total expenditures, color-coded by region (East = orange, North = yellow, West = purple). Region filter slicer on the right with summary KPIs (number of customers, total expended, total allocated) and a bar chart of vendor count by region. Tooltip on each bubble shows vendor name, address, region, and total expenditures, with drill-through to Vendor Enrollment.

### Expenses with CIP & Vendor ID
A detailed transaction-level table showing every customer's funding program, customer address, scholarship account ID, scholarship budget ID, customer name, and more — filtered to Approved budgets and Authorized requests. Useful for auditing and case-level investigation.

---

## Data Sources

| Source | Connection Method | Tables Populated |
|---|---|---|
| **Salesforce** | Direct org connector (live) | Scholarship Account, Scholarship Budget, Scholarship Request, Account, Vendor Enrollment, Vendor Location, Program Cohort, Program Engagement, Program Location, Funding Program, CIP Code/Title |
| **MIP** (financial accounting system) | Scheduled import | MIP Expenditures |
| **CSV** | Static file import | Region – ZIP |

> Salesforce objects were connected directly using an org account — no intermediate export or ETL layer. Table relationships were identified and mapped manually, as no prior documentation existed for the Salesforce object schema.

---

## Data Model

**21 tables** total. The model uses a mix of fact tables (Unified_SR_SB, MIP Expenditures), dimension tables (Account, CIP Code/Title, Region-ZIP, Fiscal Year Table), and supporting utility tables (Measure Table, Fiscal Month Axis, Refresh Info, UsedUp Filter).

### Table Reference

| Table | Type | Description |
|---|---|---|
| `Unified_SR_SB` | Calculated (DAX) | ⭐ Central fact table — union of Scholarship Requests and orphaned Scholarship Budgets |
| `Scholarship Budget` | Salesforce | Customer-level fund allocations (one per customer per program year) |
| `Scholarship Request` | Salesforce | Individual spend requests (training, support, gift card) |
| `Funding Program` | Salesforce | Program-level original budgets, MIP codes, available/consumed amounts |
| `MIP Expenditures` | MIP import | Direct expenditures from the financial accounting system |
| `Account` | Salesforce | Customer accounts with billing city, state, ZIP |
| `Vendor Enrollment` | Salesforce | Vendor-to-cohort mappings with CIP codes and scholarship account links |
| `Vendor Location` | Salesforce | Vendor physical addresses (used for map visual) |
| `Program Cohort` | Salesforce | Training cohort metadata (name, available slots, classification) |
| `Program Engagement` | Salesforce | Customer-to-program enrollment records |
| `Program Location` | Salesforce | Physical training site data with TAPO addresses |
| `CIP Code/Title` | Salesforce | Occupation classification (CIP codes, SOC codes, titles) |
| `Region – ZIP` | CSV | ZIP code to region (East/North/West) lookup |
| `Fiscal Year Table` | Calculated | FY date spine with start/end dates and FY label |
| `Fiscal Month Axis` | Calculated | Month number and short label for projection axis |
| `Measure Table` | Empty (DAX only) | All DAX measures organized in one dedicated table |
| `Refresh Info` | Calculated | Dynamic footer showing last refresh timestamp and developer name |
| `Other Funds Original Budget` | Manual | Budget amounts for non-FAMS programs (Child Care, SDF, Wagner Peyser, etc.) |
| `UsedUp Filter` | Helper | Slicer table for filtering accounts by used-up status |

---

## The Unified_SR_SB Table — Core Design Decision

This is the most important design decision in the entire model.

**The problem:** Scholarship Budgets can exist in an "Approved" state with no linked Scholarship Requests — for example, a customer account has been funded but hasn't submitted any spending requests yet. Joining only through Scholarship Request silently drops those budget records, causing:
- Understated allocated fund totals
- Undercounted customer enrollments
- Incorrect % allocated and % expensed metrics

**The solution:** A calculated DAX table using `UNION()` that combines:
1. **All Scholarship Request rows** — with their parent Scholarship Budget columns pulled in via `RELATED()`
2. **Scholarship Budget-only rows** — budgets with no matching request, where all SR columns are `BLANK()`

Each row is tagged with a `Row_Type` column (`"Request"` or `"Budget"`) and a `Unified_ID` prefix (`"SR-"` or `"SB-"`) so that measures can filter precisely.

```dax
Unified_SR_SB = 
UNION(
    // Part 1: Scholarship Request rows (with SB columns via RELATED)
    SELECTCOLUMNS(
        ADDCOLUMNS('Scholarship Request',
            "Id_SB",   RELATED('Scholarship Budget'[Id]),
            "Status__c_SB", RELATED('Scholarship Budget'[Status__c]),
            "Amount_Allocated_to_Customer_Accounts_SB",
                RELATED('Scholarship Budget'[Amount Allocated to Customer Accounts]),
            // ... all other SB columns
        ),
        "Unified_ID", "SR-" & 'Scholarship Request'[Id],
        "Row_Type",   "Request",
        // ... SR columns mapped directly
    ),

    // Part 2: Budget-only rows (no linked SR)
    SELECTCOLUMNS(
        FILTER('Scholarship Budget',
            ISBLANK(RELATED('Scholarship Request'[Id]))
        ),
        "Unified_ID", "SB-" & 'Scholarship Budget'[Id],
        "Row_Type",   "Budget",
        // SR columns all set to BLANK()
        // SB columns mapped directly
    )
)
```

This guarantees every approved budget is counted exactly once, regardless of whether requests have been submitted.

---

## DAX Measures — Key Patterns & Design Decisions

All measures live in a dedicated `Measure Table` with no data columns — keeping the model organized and all logic version-trackable.

### Preventing Double-Counting with SUMX + VALUES

Because the unified table has multiple rows per scholarship budget (one budget row + N request rows), measures that aggregate budget-level values use `SUMX(VALUES([Id_SB]), CALCULATE(MAX(...)))`. This evaluates the budget amount once per unique budget ID — preventing it from multiplying with request count.

```dax
Total Allocated FAMS v2 = 
SUMX(
    VALUES('Unified_SR_SB'[Id_SB]),
    CALCULATE(
        MAX('Unified_SR_SB'[Amount_Allocated_to_Customer_Accounts_SB]),
        'Unified_SR_SB'[Status__c_SB] = "Approved"
    )
)
```

### Two Distinct "Available" Concepts

The model distinguishes two meanings of "available" that are easy to conflate:

| Measure | Formula | Meaning |
|---|---|---|
| `Available to the System v2` | Original Budget − Allocated (FAMS) − MIP Expenditures | Unallocated pool still available to assign to new customers |
| `Available Financial Aid Budget v2` | Original Budget − All Expenditures | Remaining cash balance — actual money not yet spent |

### Capping Percentages at 100%

When expenses slightly exceed allocated amounts (timing differences between systems), raw `DIVIDE()` can return values over 1. This is capped defensively:

```dax
% Account Budget Expensed v2 = 
MIN(DIVIDE([Total Expenditures FAMS v2], [Total Allocated FAMS v2], 0), 1)
```

### Conditional Formatting via DAX Measure

Color alerts for funds approaching full obligation (>85% used) are returned as hex color strings from a measure and applied as cell background color — no separate conditional formatting rules required, and the logic stays in DAX where it's auditable:

```dax
Funds Approaching Full Obligation Color = 
VAR PctUsed = DIVIDE([Total Allocated + Expenditures MIP], [Original Budget], 0)
RETURN
    IF(ISBLANK([Original Budget]), BLANK(),
       IF(PctUsed > 0.85, "#F4CCCC", BLANK()))
```

### Fund Exhaustion Projection

The projection table estimates how many days until each fund is fully obligated based on the current daily allocation run rate, then projects whether the fund will last through the fiscal year end:

```dax
Projected Allocated per Day v2 = 
DIVIDE([Total Allocated FAMS v2], [Days Since FY Start], 0)

Projected Budget Used Up Date Raw v2 = 
[Days Left in FY] - 
DIVIDE([Available to the System v2], [Projected Allocated per Day v2], 0)
```

The Gantt heatmap colors each fiscal month cell red or green based on whether that month is at or past the projected exhaustion point.

### Top 15 with "Rest of" Complement

Vendor, CIP, and cohort ranking visuals use a consistent `TOPN` + `SUMX` pattern. A matching "Rest of" measure subtracts the Top 15 total from the full total to power the "Top 15 vs. All Others" donut charts:

```dax
Top 15 Vendor Expenditures v2 = 
SUMX(
    TOPN(15,
        FILTER(VALUES('Vendor Enrollment'[Vendor Name]),
               NOT ISBLANK('Vendor Enrollment'[Vendor Name])
               && 'Vendor Enrollment'[Vendor Name] <> ""),
        [Total Expenditures FAMS v2], DESC),
    [Total Expenditures FAMS v2]
)

Rest of Vendors Expenditures v2 = 
VAR TotalAll = CALCULATE([Total Expenditures FAMS v2],
    REMOVEFILTERS('Vendor Enrollment'),
    FILTER(ALL('Vendor Enrollment'[Vendor Name]),
           NOT ISBLANK('Vendor Enrollment'[Vendor Name])
           && 'Vendor Enrollment'[Vendor Name] <> ""))
RETURN MAX(0, TotalAll - [Top 15 Vendor Expenditures v2])
```

### Dynamic Refresh Footer

A `Refresh Info` table contains a `Dynamic Footer` measure that shows developer credit and last refresh time as a tooltip on the report header info icon:

> *Developed by Dylan Bui | Last Refreshed: 5/13/2026 5:49 AM*

---

## UX & Navigation Design

The dashboard uses **Power BI Bookmarks** to simulate multi-section navigation within pages — keeping the experience smooth without full page reloads.

**Bookmark groups:**
- `Main` → Home landing state
- `WIOA selected` / `TANF, SNAP, NDW selected` / `Others selected` → Fund category tabs
- `Home (Expenditures)` / `Expenditures Options` → Expenditure page sub-navigation
- `Occupation Expenditures` / `Provider Expenditures` / `Programs Expenditures` → CIP sub-views
- `Service Provider Exp` / `Regional Exp` / `Funds Exp` → Expenditures detail views

**Navigation elements:**
- Golden-bordered buttons for active primary navigation (Occupations/CIP, Expenditures, Projections, Enrollments, Vendor Maps)
- Back arrow to return from drill-through detail pages to the fund overview
- Right-click drill-through from bar charts → Fund Details; from vendor charts → Vendor Maps
- Information tooltip on the header (hover for developer name + refresh timestamp)
- Submit Feedback button (speech bubble icon) for end-user feedback collection

---

## Full Measure List

| Measure | Description |
|---|---|
| `% Account Budget Expensed v2` | Expenditures ÷ Allocated, capped at 100% |
| `% Available to the System v2` | Available to System ÷ Original Budget |
| `% Budget Usage v2` | (Expended + Encumbered) ÷ (Expended + Encumbered + Committed) |
| `% Fund Allocated v2` | (Original − Available) ÷ Original |
| `% Fund Expensed v2` | (FAMS + MIP Expenditures) ÷ Original Budget, capped at 100% |
| `Accounts Not Used Up Allocated v2` | Count of accounts where expenses < allocated amount |
| `Accounts Used Up Allocated v2` | Count of accounts where expenses ≥ allocated amount |
| `Amount Committed to Customer v2` | Allocated − Encumbered − Expended per budget (uncommitted remaining balance) |
| `Available Financial Aid Budget v2` | Original Budget − All Expenditures (cash remaining) |
| `Available to the System v2` | Original Budget − FAMS Allocated − MIP Expenditures (unassigned pool) |
| `Customer Balance Amount` | Allocated − Expenses per budget (individual customer balance) |
| `Days Left in FY` | Fiscal year end date − today |
| `Distinct Accounts v2` | Unique customer accounts with Approved budgets |
| `Distinct Accounts (With Requests) v2` | Unique accounts with Approved + Authorized requests |
| `Distinct Training Accounts v2` | Accounts with Training-type authorized requests |
| `Distinct Support Accounts v2` | Accounts with Support/Gift Card authorized requests |
| `Distinct Vendor Count` / `Distinct Vendor Count 2` | Vendor counts by ID vs. by name |
| `Enrollments per $1K Spent` | Distinct accounts ÷ (total expenditures ÷ 1000) |
| `Fiscal Month Start Date` / `Fiscal Month End Date` | Dynamic month boundaries based on selected FY and month number |
| `Funds Approaching Full Obligation Color` | Hex string for conditional formatting (>85% obligated = light red) |
| `Funds Approaching Full Obligation Color (Others)` | Same logic for non-FAMS funds |
| `Number of Training Requests v2` | Row count of Training-type SR rows |
| `Number of Support Requests v2` | Row count of Support/Gift Card SR rows |
| `Original Budget` | SUM of MIP Original Amount from Funding Program |
| `Original Budget (Others)` | SUM of Original Budget from Other Funds table |
| `Percentage Obligated To Date` | Actual allocated ÷ original budget (for projection table) |
| `Projected Allocated per Day v2` | Daily allocation run rate based on FY-to-date spending |
| `Projected Allocated per Month v2` | Daily rate × average days per month |
| `Projected Budget Allocated To Date` | Projected total obligation by fiscal year end |
| `Projected Budget Used Up Date v2` | Estimated date when fund will be fully obligated |
| `Projected Budget Timeline Color` | Red/green hex for Gantt heatmap cell |
| `Projected Budget Timeline Fill` | Fill opacity for Gantt cells |
| `Support Account Budget v2` | Budget allocated to Support/Gift Card type accounts |
| `Training Account Budget v2` | Budget allocated to Training type accounts |
| `Support Expenses v2` / `Training Expenses v2` | Expenditures split by program type |
| `Top 15 CIP/Vendor/Cohort Allocated v2` | Top 15 by allocated amount |
| `Top 15 CIP/Vendor/Cohort Expenditures v2` | Top 15 by expenditure amount |
| `Rest of CIP/Vendor/Cohort Allocated/Expenditures v2` | Complement of Top 15 for "All Others" donut segment |
| `Total Allocated FAMS v2` | Total approved scholarship budget allocated to customers |
| `Total Allocated Display v2` | Formatted string version ("$1.4M" / "$946K") for card visuals |
| `Total Encumbered v2` | Total encumbrances on authorized requests |
| `Total Encumbered Display v2` | Formatted string version |
| `Total Expenditures FAMS v2` | Total expenses on authorized requests |
| `Total Expenditures Display v2` | Formatted string version |
| `Total Expenditures MIP` | Direct expenditures from MIP financial system |
| `Total Expenditures FAMS + MIP v2` | Combined total from both systems |
| `Total Allocated + Expenditures MIP` | FAMS Allocated + MIP Expenditures (for obligation % calc) |
| `Unexpensed Financial Aid v2` | Original Budget − All Expenditures |
| `CIP Title Not Blank Flag` | 1/0 flag to filter out blank CIP titles in visuals |

---

## Skills Demonstrated

**Data Engineering**
- Connecting Power BI directly to Salesforce CRM objects via live org connector
- Integrating three heterogeneous data sources (Salesforce, MIP financial system, CSV) into a unified semantic model
- Manually reverse-engineering and documenting table relationships with no prior schema documentation
- Designing a `UNION()`-based calculated table to handle many-to-optional relationships without data loss

**DAX**
- Advanced iterator patterns (`SUMX` + `VALUES`) to avoid double-counting in a denormalized fact table
- Row-context vs. filter-context management across calculated columns and measures
- Dynamic date calculations for fiscal year projection and fund exhaustion date estimation
- Measure-driven conditional formatting (returning hex color strings from DAX)
- `TOPN` ranking with complement "Rest of" measures for Top 15 analysis
- `COALESCE` and `MIN(..., 1)` defensive patterns for clean output in edge cases

**Report Design**
- Bookmark-based navigation simulating a multi-section single-page app within Power BI
- Drill-through connections from overview charts to detail pages
- Azure Maps integration with expenditure-sized bubbles and regional color-coding
- Consistent design system (Workforce Solutions brand palette, golden button borders, blue header)
- Tooltip pages with contextual KPIs surfaced on hover
- Dynamic footer with last-refresh timestamp surfaced via DAX and tooltip

**Domain Knowledge**
- Federal workforce development programs: WIOA Title I Adult/DLW/Youth, TANF, SNAP E&T ABAWD, NDW disaster recovery grants
- Scholarship fund lifecycle: Budget Approval → Request Authorization → Encumbrance → Expenditure
- Fund obligation tracking and fiscal year burn rate analysis for compliance reporting
