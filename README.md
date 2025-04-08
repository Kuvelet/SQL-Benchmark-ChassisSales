# Benchmarking Auto Parts Sales with SQL-Based Market Comparison

### Table of Contents  
- [Project Overview](#project-overview)  
- [Objectives](#objectives)  
- [Data Sources](#data-sources)  
- [Tools](#tools)  
- [Data Preparation](#data-preparation)  
- [SQL Methodology](#sql-methodology)  
- [Benchmarking & Opportunity Analysis](#benchmarking--opportunity-analysis)  
- [KPIs & Pseudo Analysis](#kpis--pseudo-analysis)  
- [Next Steps](#next-steps)

---

### Project Overview

In the global auto parts industry, wholesale suppliers face fierce competition and rapidly shifting demands. This project focuses on leveraging SQL to benchmark internal sales performance against market-wide demand trends using region-specific sales data from retailers. The goal is to uncover gaps and opportunities by aligning company sales performance with forecasted or actual sales data across key regions including North America, Central America, South America, Europe, Africa, and the Middle East.

Given the diversity in retailers' catalogs—where different brands like Moog, Mevotech, Dorman, MAS, and Delphi dominate—cross-referencing part numbers becomes essential. By maintaining a robust Master Cross, we translate competitor part numbers and OEM references into our own catalog, ensuring accurate comparisons.

---

### Objectives

- Normalize and integrate inconsistent retailer data for meaningful regional comparison  
- Cross-reference competitor part numbers to internal SKUs using a dynamic catalog system  
- Benchmark our sales data against forecasted or actual demand to:  
  - Identify product coverage gaps  
  - Measure competitive presence  
  - Uncover regional growth opportunities  
- Present findings via SQL outputs and internal Power BI dashboards (data only; no visuals)

---

### Data Sources

- **Retailer Demand Files** (varies in format by source)  
  - Metrics: `Qty Sold`, `Brand`, `Description`, `Region`, `Date`  
  - Frequency: Varies (6M, 12M, 18M) — normalized to annual values
- **Internal Sales Database** (queried via Power BI from SQL Server)  
- **Cross Reference Master Table**  
  - Columns: `OEM Number`, `Company Part#`, `Competitor Crosses` (Moog, Delphi, etc.)

---

### Tools

- **SQL Server (T-SQL)**: Data cleaning, transformation, aggregation  
- **Power BI**: Used solely to retrieve internal sales figures for integration

---

### Data Preparation

Retailer demand files arrive in various structures and must be manually mapped. Key challenges include:

- **Inconsistent Part Numbers**: Dashes, dots, and spaces are removed to standardize formats  
- **Cross-Matching**: Competing brand part numbers are mapped to internal SKUs using `Master_CrossReferences`  
- **Brand Filtering**: Only OE-grade aftermarket parts are analyzed; economy brands are excluded  
- **Region Labeling**: Region and date fields are manually appended if missing

> *Note: Superseded or legacy part numbers are updated manually in the Master Cross when discrepancies are found.*

---

### SQL Methodology

1. **Standardization**  
   - Condensed versions of part numbers created (e.g., removing special characters)  
   - Dates normalized to represent annual values (i.e., extrapolated from shorter or longer periods)

2. **Cross-Matching Process**  
   - Competitor part numbers joined with Master Cross table to resolve to our SKU  
   - If multiple matches occur, preference is given to OE-grade items  
   - Duplicates caused by multi-brand overlaps are aggregated

3. **Demand Aggregation**  
   - Demand values grouped by region and part  
   - Cross-referenced sales quantities summed up for each SKU

4. **Sales Integration**  
   - Internal annual sales pulled from Power BI (linked to SQL)  
   - Merged into the benchmark table using our part number as key

---

### Benchmarking & Opportunity Analysis

Once normalized and aligned, each part-region pair is analyzed for:

- **Sales vs Demand Delta**  
  - Positive delta = Growth achieved  
  - Negative delta = Potential missed opportunity

- **Market Presence**  
  - Presence in catalogs without sales  
  - Sales where no catalog demand was detected (indicating possible distribution gaps)

- **Brand Positioning Check**  
  - Comparing internal product against the type and tier of competitors in that region

---

### KPIs & Pseudo Analysis

#### 🔸 1. Lost Opportunity %

(Lost Sales / Total Demand) * 100

**Example**:  
- Region: North America  
- SKU: 123456  
- Demand: 12,000 units  
- Our Sales: 4,200 units  
- **Lost Opportunity** = (12,000 - 4,200) / 12,000 = **65%**

#### 🔸 2. Brand Penetration Rate

(Our Sales / Total Regional Demand for SKUs we offer)

**Example**:  
- Region: South America  
- Total Demand: 25,000 units  
- Our Sales: 8,750 units  
- **Penetration** = 35%

#### 🔸 3. Catalog Coverage Ratio

(Unique SKUs Sold / Unique SKUs with Demand)

**Example**:  
- SKUs with demand: 950  
- SKUs we sold: 440  
- **Coverage** = 46.3%

#### 🔸 4. Fill Rate Proxy

(Sales / Min(Sales, Demand)) [capped at 100%]

**Example**:  
- SKU 789012 in Middle East  
- Demand: 3,000 units  
- Sales: 2,750 units  
- **Fill Rate** ≈ 91.7%

---

### Next Steps

1. **Automate Matching Rules**  
   - Introduce rule-based suppression for economy crosses  
   - Automate detection of mismatched or multi-matched competitor SKUs

2. **Advanced Metrics**  
   - Develop weighted metrics that account for margin, freight, and tax variables by region

3. **Gap-Focused Alerts**  
   - Flag parts with demand but no sales in a region for follow-up by regional managers

4. **Improve Forecast Validation**  
   - Compare forecasted vs actual sales to tune demand reliability per retailer

---

> *This SQL-driven benchmarking framework provides the sales and product strategy teams with a structured way to monitor performance and identify regional growth opportunities based on live market demand.*
