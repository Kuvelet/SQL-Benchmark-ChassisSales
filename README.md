# Benchmarking Auto Parts Sales with SQL-Based Market Comparison

### Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Data Sources](#data-sources)
- [Tools](#tools)
- [Data Preparation](#data-preparation)
- [SQL Methodology](#sql-methodology)
- [SQL Procedures & Data Flow Explanation](#sql-procedures--data-flow-explanation)
  - [Data Cleaning](#data-cleaning)
  - [sp_CreateMasterCrossReferencesCK](#sp_createmastercrossreferencesck)
    - [Example Transformation](#example-transformation)
  - [Update_Chassis_Rank](#update_chassis_rank)
    - [Example Transformation](#example-transformation-1)
- [Business Use & Workflow Integration](#business-use--workflow-integration)
- [Benchmarking Analysis](#benchmarking-analysis)
  - [Scenario Overview](#scenario-overview)
  - [Aggregated Retailer Demand](#aggregated-retailer-demand-annualized)
  - [Internal Sales Performance](#internal-sales-performance)
  - [Opportunity & Penetration Metrics](#opportunity--penetration-metrics)
  - [Observations & Insights](#observations--insights)
  - [Strategic Actions](#strategic-actions)
- [Catalog Coverage Analysis](#catalog-coverage-analysis)
  - [Catalog Coverage Definition](#catalog-coverage-definition)
  - [Sample Regional Coverage Snapshot](#sample-regional-coverage-snapshot)
  - [Observations & Strategic Insights](#observations--strategic-insights)
  - [Action Plan for Catalog Alignment](#action-plan-for-catalog-alignment)
- [Next Steps](#next-steps)

---

### Project Overview

In the global auto parts industry, wholesale suppliers operate within a highly competitive and fragmented marketplace, where product demand, brand preferences, and pricing strategies vary dramatically by region. This project leverages SQL to benchmark our internal sales performance against regional market demand, enabling a deeper understanding of how well our product lines perform in each geographic area—including North America, Central America, South America, Europe, Africa, and the Middle East.

By mapping retailer demand data to our internal catalog using a dynamic Master Cross system, we are able to translate a wide variety of OEM and competitor part numbers (e.g., Moog, Mevotech, Dorman, MAS, Delphi) into our internal identifiers. This allows us to perform robust, SKU-level comparisons between our actual sales and the total market opportunity. Through this analysis, we can:

Measure market penetration for each part in each region

Identify coverage gaps where demand exists but our product was not sold

Recognize saturation zones where our sales align well with demand

Support strategic decisions on pricing, stocking, and catalog development based on real-world market behavior

Ultimately, the goal is to better understand regional market dynamics and improve our competitiveness by continuously adapting our offering to what each market truly demands.

---


### Objectives

- **Normalize and integrate** inconsistent, retailer-sourced sales and forecast data to create a unified foundation for market comparison  
- **Cross-reference** diverse competitor part numbers and OEM references against internal SKUs using a dynamic Master Cross system  
- **Benchmark our sales** performance against total regional demand in order to:  
  - Identify **underperforming SKUs** where sales exist but significantly trail behind demand  
  - Detect **coverage gaps** where demand exists but no sales were captured  
  - Measure **market penetration levels** across key regions and brands  
  - Pinpoint **strategic growth opportunities** in underserved areas  
- Support sales and catalog decisions by delivering results through **SQL-based outputs** and **Power BI live dashboards** connected to cleaned, structured data


---


### Data Sources

- **Retailer Demand Files** (format varies by source)  
  These are Excel-based sales or forecast datasets collected externally by our sales representatives from retail partners.  
  - Common Metrics: `Qty Sold`, `Brand`, `Description`, `Region`, `Date`  
  - Timeframes: Varies (6M, 12M, 18M); all values are normalized to reflect **annual demand**

- **Internal Sales Data**  
  Internal sales performance data is accessed **directly within Power BI**, where it is maintained and refreshed.  
  - Not hosted on SQL Server  
  - Used in **live benchmarking analysis** by joining Power BI results to SQL outputs

- **Cross Reference Master Table**  
  The master catalog (`01_CrossMaster`) contains mappings between our internal SKUs and equivalent part numbers from OEMs and aftermarket brands (e.g., Moog, Delphi, MAS, Mevotech).  
  - Structure:  
    - `SusCatalog` (Internal Part Number)  
    - `OEM Number`  
    - Dozens of **brand-specific columns**, one for each competitor (e.g., `Moog`, `MevotechOG`, `DormanCond`, etc.)

  Because these crosses are spread across columns, we use the stored procedure `sp_CreateMasterCrossReferencesCK` to **flatten** the table into a clean, queryable structure:
  - `CrossName` – the brand column name  
  - `CrossingNumber` – the competitor or OEM part number  
  - `SusCatalog` – our internal part number

  This flattened structure enables efficient and consistent joins between external retailer data and internal catalog identifiers.

---

### Tools

- **SQL Server (T-SQL)**: Data cleaning, transformation, aggregation  
- **Power BI**: Used solely to retrieve internal sales figures for integration

---

### Data Preparation

Retailer demand files are received in a variety of formats and structures, depending on the source. Each file must be carefully cleaned and standardized before it can be integrated into the benchmarking model. Key steps and challenges include:

- **Inconsistent Part Numbers**  
  Part numbers often contain dashes, dots, extra spaces, or inconsistent casing. These are cleaned to generate **condensed versions** used for reliable matching.

- **Cross-Matching to Internal Catalog**  
  Cleaned part numbers from retailer files are joined to our internal SKUs using the `Master_CrossReferences_CK` table. This allows us to identify the corresponding internal `SusCatalog` for each external part reference.

- **Brand Filtering**  
  Only **OE-grade** aftermarket brands are included in the analysis. Budget or economy-tier brands are excluded to maintain consistency with our product positioning.

- **Region and Date Attribution**  
  Retailer files may lack explicit region or time period fields. These are manually inferred and appended based on the source of the file or accompanying metadata.

> **Note:** In cases where multiple external part numbers map to the same internal SKU, deduplication is handled through row-level filtering. Superseded or incorrect catalog crosses are updated manually in the master cross-reference table to ensure alignment across all projects.

---

### SQL Methodology

1. **Standardization**  
   - All part numbers are transformed into **condensed formats** by removing special characters (e.g., dashes, dots, spaces) to ensure consistent matching  
   - Sales data timeframes (6M, 12M, 18M) are **normalized to annual values** to enable fair, apples-to-apples comparison across sources

2. **Cross-Matching Process**  
   - Cleaned part numbers from retailer files are joined to the flattened `Master_CrossReferences_CK` table  
   - This resolves each external part number to our internal `SusCatalog` value  
   - If multiple matches are found due to overlap or format collisions, only one is retained using `ROW_NUMBER()` filtering  
   - This avoids double-counting and ensures that each retailer part number maps to a **single internal SKU**

3. **Demand Aggregation**  
   - After matching, all retailer demand data is **grouped by region and internal SKU**  
   - Aggregated quantities are calculated to reflect **total external demand** for each part-region pair

4. **Sales Integration**  
   - Internal sales performance is retrieved directly from **Power BI**  
   - These sales figures are **joined to the demand table** using our part number (`SusCatalog`) as the key  
   - This merged dataset enables precise benchmarking, gap analysis, and opportunity scoring

---


### SQL Procedures & Data Flow Explanation

This section documents the core SQL procedures used in the benchmarking workflow. These procedures are vital for transforming, unifying, and analyzing external sales data from retailers and internal catalogs.

#### Data Cleaning

After compiling and importing the raw sales data from global sales representatives into the SQL Server table `ALL_CHASSIS_SALES_RAW`, a preprocessing step is performed to clean part numbers and prepare the dataset for accurate cross-referencing.

- **Standardize Part Numbers**:  
  Raw part numbers often contain inconsistent formatting such as dashes, dots, slashes, or spaces. These characters are removed to generate a clean, condensed version.

- **Create `Part_Number_Cond` Field**:  
  A new column, `Part_Number_Cond`, is generated to store the cleaned version of each part number. This ensures that all matching and joining operations (particularly with the Master Cross Reference table) are based on consistent part number formatting.

```sql
SELECT
    ID,
    Brand,
    Part_Number,
    UPPER(REPLACE(REPLACE(REPLACE(REPLACE([Part_Number], '-', ''), '.', ''), ' ', ''), '/', '')) AS Part_Number_Cond,
    Last_12_Months_Sales,
    List,
    Region,
    Year_Received
INTO dbo.ALL_CHASSIS_SALES_RAW_CLEANED
FROM dbo.ALL_CHASSIS_SALES_RAW;
```

##### Example Transformation:

| Part_Number   | Part_Number_Cond |
|---------------|------------------|
| `1AS-BJ00132` | `1ASBJ00132`     |
| `TRQ.123 / A` | `TRQ123A`        |
| ` 123-456 `   | `123456`         |



#### `[sp_CreateMasterCrossReferencesCK]`  

**Purpose**:  
Transform the wide-format cross-reference catalog (`01_CrossMaster`) into a **flattened, vertically structured table** (`Master_CrossReferences_CK`) to support efficient and scalable joins between competitor part numbers and our internal catalog.

Retailer demand files often include competitor part numbers (e.g., from Moog, Delphi, MAS, etc.), but in the source catalog, each competitor's part number is stored in its own dedicated column. This stored procedure restructures the catalog into a unified reference table with the following columns:

- `CrossName` → The name of the original brand column (e.g., `Moog`, `DelphiCond`)  
- `CrossingNumber` → The competitor or OEM part number  
- `SusCatalog` → Our internal part number used across all analysis

Using a `UNION ALL` approach, this procedure vertically stacks all relevant columns from `01_CrossMaster` into a single table. It also filters out nulls to maintain clean and reliable joins.

The resulting `Master_CrossReferences_CK` table supports flexible cross-matching logic, whether for market benchmarking, catalog alignment, or sales performance analysis. Conditional (`Cond`) and related-brand fields are also included to maximize compatibility across all use cases, even if not all brands are used in every analysis.

This procedure is executed **on demand**, typically just before running a new benchmarking or sales comparison cycle.

>  *Note:* All part numbers used in matching are cleaned to a condensed format (removing dashes, dots, and spaces) to ensure consistent joins and prevent formatting mismatches.

```sql
IF OBJECT_ID('dbo.Master_CrossReferences_CK', 'U') IS NOT NULL
    DROP TABLE dbo.Master_CrossReferences_CK;

SELECT DISTINCT t.CrossName, t.CrossingNumber, t.SusCatalog
INTO dbo.Master_CrossReferences_CK
FROM (
    SELECT 'OEMCond' AS CrossName, OEMCond AS CrossingNumber, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
    UNION ALL
    SELECT 'Moog', Moog, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
    UNION ALL
    SELECT 'Delphi', Delphi, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
    -- +40 additional brand columns in full implementation
    UNION ALL
    SELECT 'Sankei', Sankei, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
) AS t
WHERE t.CrossingNumber IS NOT NULL AND t.SusCatalog IS NOT NULL;
```

#### Example Transformation

The original `01_CrossMaster` table stores competitor part numbers in a **wide format**, where each column represents a different brand:

| SusCatalog | Moog     | Delphi   | MAS       | OEMCond   |
|------------|----------|----------|-----------|-----------|
| SUS-10001  | K123456  | DS45678  | MS98765   | 12345678  |
| SUS-10002  | K789012  | NULL     | MS65432   | 87654321  |

The stored procedure `[sp_CreateMasterCrossReferencesCK]` transforms this into a **long (tall)** format for efficient querying and cross-referencing:

| CrossName | CrossingNumber | SusCatalog |
|-----------|----------------|------------|
| Moog      | K123456        | SUS-10001  |
| Delphi    | DS45678        | SUS-10001  |
| MAS       | MS98765        | SUS-10001  |
| OEMCond   | 12345678       | SUS-10001  |
| Moog      | K789012        | SUS-10002  |
| MAS       | MS65432        | SUS-10002  |
| OEMCond   | 87654321       | SUS-10002  |

This flattened format enables us to **match any part number, from any brand**, against our catalog without having to search across multiple columns.


### 2. `[Update_Chassis_Rank]`  
**Purpose**: Aggregate retailer sales data by region, resolve crosses to internal part numbers, and rank performance by part and geography.

- `ALL_CHASSIS_SALES_RAW`: Cleaned and compiled from Excel files submitted by sales reps.
- This table includes:  
  `Part_Number`, `Brand`, `Qty`, `Region`, `Date`, `Sales`, and `Part_Number_Cond` for normalized comparisons.


####  1st Part: Cross Reference Matching
- Joins `ALL_CHASSIS_SALES_RAW` with `Master_CrossReferences_CK` using the `Part_Number_Cond` field.
- Uses `ROW_NUMBER()` to deduplicate multiple matches for the same part by selecting only the first record.
  - This avoids **double aggregation** of sales data.
  - Ensures one-to-one mapping when multiple brands point to the same internal SKU.

#### 2nd Part: Regional Aggregation & Ranking
- Aggregates sales by `SusCatalog` across all major regions:
  - North America
  - Mexico
  - Puerto Rico
  - Europe
  - Africa
  - Central America
  - South America
  - Middle East
- Computes **Total Sales** and region-specific subtotals.
- Applies `RANK()` function per region to create region-based leaderboard metrics for each internal part number.

```sql
ALTER PROCEDURE [dbo].[Update_Chassis_Rank]
AS
BEGIN
    SET NOCOUNT ON;

    IF OBJECT_ID('CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed', 'U') IS NOT NULL
        DROP TABLE CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed;

    SELECT 
        A.ID, A.Brand, A.Part_Number, A.Part_Number_Cond,
        A.Last_12_Months_Sales, A.List, A.Region, A.Year_Received,
        F.CrossName, F.CrossingNumber, F.SusCatalog
    INTO CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed
    FROM CAN.dbo.ALL_CHASSIS_SALES_RAW A
    LEFT JOIN (
        SELECT CrossName, CrossingNumber, SusCatalog
        FROM (
            SELECT *, ROW_NUMBER() OVER (PARTITION BY CrossingNumber ORDER BY CrossName) AS row_num
            FROM CAN.dbo.Master_CrossReferences_CK
        ) R
        WHERE row_num = 1
    ) F ON A.Part_Number_Cond = F.CrossingNumber;

    IF OBJECT_ID('CAN.dbo.Chassis_Rank', 'U') IS NOT NULL
        DROP TABLE CAN.dbo.Chassis_Rank;

    ;WITH RegionalSales AS (
        SELECT 
            SusCatalog,
            SUM(CASE WHEN Region = 'North America' THEN Last_12_Months_Sales ELSE NULL END) AS NA_Sales,
            SUM(CASE WHEN Region = 'Mexico' THEN Last_12_Months_Sales ELSE NULL END) AS Mexico_Sales,
            SUM(CASE WHEN Region = 'Puerto Rico' THEN Last_12_Months_Sales ELSE NULL END) AS PuertoRico_Sales,
            SUM(CASE WHEN Region = 'Europe' THEN Last_12_Months_Sales ELSE NULL END) AS Europe_Sales,
            SUM(CASE WHEN Region = 'Africa' THEN Last_12_Months_Sales ELSE NULL END) AS Africa_Sales,
            SUM(CASE WHEN Region = 'Central America' THEN Last_12_Months_Sales ELSE NULL END) AS Central_Sales,
            SUM(CASE WHEN Region = 'South America' THEN Last_12_Months_Sales ELSE NULL END) AS SA_Sales,
            SUM(CASE WHEN Region = 'Middle East' THEN Last_12_Months_Sales ELSE NULL END) AS ME_Sales,
            SUM(Last_12_Months_Sales) AS Total_Sales
        FROM CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed
        WHERE SusCatalog IS NOT NULL
        GROUP BY SusCatalog
    )
    SELECT *, 
        RANK() OVER (ORDER BY Total_Sales DESC) AS Rank_Total
    INTO CAN.dbo.Chassis_Rank
    FROM RegionalSales;
END;
```

#### Example Transformation

**Raw Input** (`ALL_CHASSIS_SALES_RAW` — sample)

| ID | Brand  | Part_Number | Part_Number_Cond | Last_12_Months_Sales | Region          | Year_Received |
|----|--------|--------------|------------------|------------------------|------------------|----------------|
| 1  | Moog   | K123456      | K123456          | 1200                   | North America    | 2023           |
| 2  | MAS    | MS98765      | MS98765          | 600                    | Mexico           | 2023           |
| 3  | Delphi | DS45678      | DS45678          | 900                    | Europe           | 2023           |
| 4  | OEM    | 12345678     | 12345678         | 1500                   | North America    | 2023           |
| 5  | Moog   | K789012      | K789012          | 500                    | South America    | 2023           |
| 6  | MAS    | MS65432      | MS65432          | 300                    | Central America  | 2023           |
| 7  | Delphi | DS11111      | DS11111          | 700                    | Mexico           | 2023           |
| 8  | OEM    | 87654321     | 87654321         | 1000                   | Europe           | 2023           |
| 9  | Moog   | K222333      | K222333          | 450                    | Middle East      | 2023           |


**Cross-Matched** (`Master_CrossReferences_CK` — simplified sample)

| CrossName | CrossingNumber | SusCatalog  |
|-----------|----------------|-------------|
| Moog      | K123456        | SUS-10001   |
| MAS       | MS98765        | SUS-10001   |
| Delphi    | DS45678        | SUS-10001   |
| OEMCond   | 12345678       | SUS-10001   |
| Moog      | K789012        | SUS-10002   |
| MAS       | MS65432        | SUS-10002   |
| Delphi    | DS11111        | SUS-10003   |
| OEMCond   | 87654321       | SUS-10003   |
| Moog      | K222333        | SUS-10004   |


**Final Output** (`Chassis_Rank` — Aggregated & Ranked)

| SusCatalog | NA_Sales | Mexico_Sales | Europe_Sales | Central_Sales | SA_Sales | ME_Sales | Total_Sales | NA_Rank | Mexico_Rank | Europe_Rank | Rank_Total |
|------------|----------|--------------|--------------|----------------|----------|----------|--------------|---------|-------------|-------------|------------|
| SUS-10001  | 2700     | 600          | 900          | NULL           | NULL     | NULL     | 4200         | 1       | 2           | 2           | 1          |
| SUS-10003  | NULL     | 700          | 1000         | NULL           | NULL     | NULL     | 1700         | NULL    | 1           | 1           | 2          |
| SUS-10002  | NULL     | NULL         | NULL         | 300            | 500      | NULL     | 800          | NULL    | NULL        | NULL        | 3          |
| SUS-10004  | NULL     | NULL         | NULL         | NULL           | NULL     | 450      | 450          | NULL    | NULL        | NULL        | 4          |

Each part number is ranked based on its **regional performance** (`RANK() OVER (ORDER BY Region_Sales DESC)`) as well as total sales across all regions. This structured output serves as the foundation for a wide range of business insights — enabling teams to identify high-performing SKUs in each market, highlight underperforming or unrepresented parts, and evaluate regional competitiveness.

By surfacing these trends, the final ranked table supports:
- Region-specific **performance dashboards**
- Strategic **gap analysis** (e.g., no sales in high-demand regions)
- **Part-level benchmarking** for pricing, stocking, and catalog development
- Smarter **territory planning** and inventory allocation across global regions

---

### Business Use & Workflow Integration

The final output table, `Chassis_Rank`, serves as a core dataset for regional sales analysis and strategic planning. It is used directly in:

- **Power BI dashboards** (via live connection)
- **MS Access queries** for daily business operations

Typical workflow:

- Run `sp_CreateMasterCrossReferencesCK` to create the latest flattened mapping between competitor part numbers and internal SKUs.
  
- Load compiled retailer sales into `ALL_CHASSIS_SALES_RAW`. Clean part numbers and normalize sales values to annual equivalents.
  
- Run `Update_Chassis_Rank` to aggregate demand by region and rank internal parts by market performance.

- Use the output in Power BI or MS Access to perform gap analysis, identify growth opportunities, and guide inventory and sales strategy.

These procedures form the technical foundation for transforming complex, retailer-specific part number data into a powerful, regionally segmented benchmarking tool.

---

###  Benchmarking Analysis

To illustrate the value of this project, below is a sample analysis using cross-referenced demand data from global retailers and our internal sales performance by region.

#### Scenario Overview

Let’s consider three internal part numbers (`SusCatalog`):

| SusCatalog | Description                   |
|------------|-------------------------------|
| SUS-10001  | Front Lower Control Arm – LH  |
| SUS-10002  | Rear Stabilizer Link – RH     |
| SUS-10003  | Upper Ball Joint Assembly     |


#### Aggregated Retailer Demand (Annualized)

| SusCatalog | North America | Mexico | Europe | Africa | ME | Total Demand |
|------------|----------------|--------|--------|--------|----|---------------|
| SUS-10001  | 2,700           | 600    | 900    | 0      | 0  | 4,200         |
| SUS-10002  | 0               | 0      | 0      | 300    | 500| 800           |
| SUS-10003  | 0               | 700    | 1,000  | 0      | 0  | 1,700         |


#### Internal Sales Performance

| SusCatalog | Internal Sales |
|------------|----------------|
| SUS-10001  | 3,900          |
| SUS-10002  | 250            |
| SUS-10003  | 300            |


#### Opportunity & Penetration Metrics

| SusCatalog | Total Demand | Internal Sales | Fulfillment Rate | Opportunity Gap |
|------------|---------------|----------------|------------------|-----------------|
| SUS-10001  | 4,200          | 3,900          | 92.9%            | 300 units       |
| SUS-10002  | 800            | 250            | 31.3%            | 550 units       |
| SUS-10003  | 1,700          | 300            | 17.6%            | 1,400 units     |


#### Observations & Insights

- `SUS-10001` shows **strong market penetration**, especially in North America and Europe. Minor opportunity remains in Mexico (~300 units), likely addressable through sales engagement or local promotions.

- `SUS-10002` underperforms despite visible demand in Africa and the Middle East. This may be due to **regional pricing misalignment** or limited distribution in those territories. It is a good candidate for expanded availability or pricing review.

- `SUS-10003` has **high demand but poor sales**, signaling a potential **stocking issue**, **catalog omission**, or again, **uncompetitive regional pricing**. It should be prioritized for corrective action such as reintroducing into local listings, bundling strategies, or adjusting landed cost structures for underserved markets.


#### Strategic Actions

- **North America**: Focus on margin optimization for top movers like `SUS-10001`.
- **Africa & ME**: Improve coverage for SKUs like `SUS-10002` through targeted promotion or new distributor onboarding.
- **Europe & Mexico**: Address `SUS-10003` shortfall with bundled offers or inclusion in more RFQs.

---


### Catalog Coverage Analysis

In addition to analyzing sales vs. demand performance, we also evaluate **catalog coverage** — measuring how many competitor-demanded SKUs are represented in our internal product line. This ensures that our product catalog is aligned with what the market is actively requesting.

#### Catalog Coverage Definition

For any given region:
- **Covered**: The competitor part number successfully cross-references to a valid internal `SusCatalog`.
- **Uncovered**: The part number appears in demand files but does **not** match any item in our Master Cross Reference, indicating a **catalog gap**.


#### Sample Regional Coverage Snapshot

| Region         | Total Competitor SKUs | Covered by Our Catalog | Uncovered (Gaps) | Coverage % |
|----------------|------------------------|-------------------------|------------------|-------------|
| North America  | 1,200                  | 1,080                   | 120              | 90.0%       |
| Mexico         | 430                    | 310                     | 120              | 72.1%       |
| Europe         | 650                    | 400                     | 250              | 61.5%       |
| Africa         | 200                    | 75                      | 125              | 37.5%       |
| Middle East    | 180                    | 100                     | 80               | 55.6%       |


#### Observations & Strategic Insights

- **North America** has excellent catalog alignment, with 90% of requested SKUs already available. Focus here should shift to margin improvement and sales velocity.
- **Mexico and Europe** show moderate coverage (~60–70%), suggesting room for **SKU expansion**, especially for fast-moving demand outside our current offering.
- **Africa and the Middle East** reflect low coverage, possibly due to **limited catalog localization**, **regional certification issues**, or delayed adoption of newer parts. These are **prime candidates** for catalog enrichment or territory-specific product bundles.


#### Action Plan for Catalog Alignment

- **Cross-Master Updates**: Review and update cross-reference logic to ensure new competitor parts are mapped where appropriate.
- **Product Development Feedback**: Share uncovered part demand with the product team to assess feasibility of onboarding missing SKUs.
- **Regional Prioritization**: Focus SKU expansion efforts on high-volume uncovered items in mid-coverage markets (e.g., Mexico, Europe).

> Regularly reviewing catalog coverage ensures that sales teams are quoting with confidence and maximizing conversion opportunities — even when the original RFQs come in under foreign brands or outdated OEM formats.

---

### Next Steps

To further enhance the accuracy, scalability, and strategic impact of this benchmarking framework, the following initiatives are recommended:

#### Expand Regional Insights
- Incorporate **retailer-level granularity** to distinguish between high- and low-value accounts
- Add **country-level breakdowns** within broad regions for deeper segmentation
- Track **regional price discrepancies** and their impact on sales performance

#### Catalog Intelligence
- Develop a flagging system to highlight **high-demand parts with no catalog entry**
- Create feedback loops to notify product and catalog teams about repeated part number gaps
- Review catalog coverage quarterly to ensure responsiveness to market shifts

#### Forecast Validation
- Compare **forecasted demand** vs. **actual sales** over time to gauge forecast reliability
- Build a model to score retailer data quality and predictive value

---

> By addressing these steps, the team can build a fully integrated ecosystem where market intelligence continuously shapes catalog decisions, regional strategies, and sales execution.
