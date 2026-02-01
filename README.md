# Technical Architecture Document
## Real Estate Market Analysis in France (January 2026)

---

## 1. Purpose of This Document

This Technical Architecture Document describes the design and implementation of a local data platform for French real estate market analysis using open government data and open-source tools.

The document explains:
- How data flows through the system
- How each component works
- Why specific technologies were chosen

This document is used to communicate the architecture clearly to technical and non-technical stakeholders.

---

## 2. Why This Document Is Important

In real companies, an architecture document is critical because it:
- Provides a global vision of the system
- Explains data flow from source to analytics
- Helps new team members understand the platform quickly
- Avoids technical confusion and duplication
- Supports maintenance and future evolution

Without an architecture document, systems become difficult to understand, maintain, and scale.

---

## 3. Who Creates This Document in a Company

In a professional environment, this document is usually created by:
- A Data Architect
- A Solution Architect
- A Lead Data Engineer

These roles are responsible for defining technical standards and ensuring that data systems are reliable, scalable, and well-structured.

---

## 4. Project Overview

### Project Name
**Real Estate Market Data Platform (French DVF)**

### Objective
The objective of this project is to design a simple but realistic data architecture that follows modern data engineering principles:
- Ingest open government data (French DVF - Direction de la Validation Foncière)
- Store raw data safely
- Clean and transform data with hierarchical imputation
- Load data into a Data Warehouse
- Enable SQL-based analytics for market insights

### Business Context
This project analyzes the French real estate market from July 2020 to June 2025, answering key business questions:
- Is the market increasing, decreasing, or stable?
- Are the most expensive departments also the most active?
- What regional disparities exist in pricing and transaction volume?
- How do apartment and house prices compare over time?

---

## 5. Global Architecture Overview

The architecture is based on a **layered approach**, which separates responsibilities and improves clarity.

The system is composed of four main layers:
1. **Data Source**
2. **Data Lake**
3. **Data Warehouse**
4. **Analytics and BI**

Each layer has a specific role and responsibility.

```
Data Source (DVF)
    ↓
Data Lake (RAW → STAGING → CURATED)
    ↓
Data Warehouse (DuckDB)
    ↓
Analytics & BI (SQL Queries)
    ↓
Outputs (CSV Reports)
```

The **Data Lake** stores data as files and preserves data flexibility and history. We use:
- **Local file system** for storage
- **CSV files** for data format
- **Python** for processing

---

## STEP 1 – Data Ingestion (RAW Zone)

### Purpose
The RAW Zone stores original data with no transformation. It acts as the source of truth.

### Data Sources
The data sources are open **DVF (Direction de la Validation Foncière) datasets** provided as CSV files.

### Characteristics
- **Public and free**: Government-provided open data
- **Static dataset**: Historical snapshot of real estate transactions
- **Represents real company data**: Realistic data quality and structure

### Role
The data source represents the entry point of the data pipeline.

### Storage Location
```
data_lake/raw/
  ├── stats_dvf.csv (monthly aggregated statistics)
  └── stats_whole.csv (global aggregates)
```

### Data Characteristics
- Monthly aggregated statistics (July 2020 - June 2025)
- Geographic hierarchies (communes, EPCI, departments, regions)
- Price metrics: median and mean €/m² for apartments and houses
- Transaction counts by property type
- ~36,000 geographic locations

---

## STEP 2 – Data Cleaning (STAGING Zone)

### Purpose
We clean and standardize data to prepare it for analysis.

### Cleaning Rules Applied

#### Temporal and Geographic Keys
- **Keep**: `annee_mois` (time identifier) and geographic keys: `code_geo`, `libelle_geo`, `code_parent`, `echelle_geo`
- **Time Format**: YYYY-MM format for consistency

#### Transaction Count Columns (`nb_ventes_*`)
- **NA → 0**: Missing values mean no sales occurred
- **Rationale**: If transaction count is missing, treat as zero transactions

#### Price Columns (`moy_prix_m2_*`, `med_prix_m2_*`)
- **If count == 0**: Set price to NA (cannot estimate when no transactions)
- **If count > 0 AND price is NA**: 
  - Impute by **median at parent geographic level** (`code_parent`)
  - Fallback to median at current level (`code_geo`) if parent unavailable
  - Add `{col}_imputed` flag to track imputation
  
#### Geographic Names (`libelle_geo`)
- **Fill missing names** with `'Inconnu'` (Unknown) if not recoverable

#### Column Reduction
- **Drop columns** with **>80% missing values**
- **Rationale**: Columns with too many missing values provide minimal analytical value

### Storage Location
```
data_lake/staging/
  ├── clean_monthly.csv (processed monthly statistics)
  └── clean_aggregated.csv (processed national aggregates)
```

### Quality Flags Added
- `{col}_imputed`: Boolean flag (TRUE if price was imputed, FALSE if real data)
- Allows analysts to filter by data confidence level

### Output of Step 2
Cleaned, standardized data ready for BI preparation with documented data quality.

---

## STEP 3 – BI Preparation (CURATED Zone)

### Purpose
Prepare **BI-ready data** for analytics and reporting.

### Characteristics
- **BI-ready data**: Fully processed and aggregated
- **Selected columns only**: Relevant metrics for analysis
- **Aggregated datasets**: Grouped to meaningful geographic and temporal levels
- **Optimized for analytics**: No further transformation needed

### Processing Logic

#### Geographic Deduplication
- **Remove EPCI** (inter-communal groupings) when child communes exist
- **Prevent double-counting** in hierarchical data
- **Example**: Remove "Métropole du Grand Paris" if Paris + child communes present

#### Level Filtering
- **Filter to department level** for final rankings
- **Exclude**: Cadastral sections, communes, EPCI for top 20 analysis
- **Rationale**: Department provides meaningful aggregation without excessive detail

#### Aggregation Strategy
- **Monthly time series**: Aggregate prices and counts by `annee_mois` and geography
- **Remove imputation flags**: No longer needed in curated layer

### Storage Location
```
data_lake/curated/
  ├── monthly_indicators.csv (time series by geography)
  ├── top20_by_transactions.csv (top 20 most active departments)
  └── top20_by_median_price.csv (top 20 most expensive departments)
```

### Output of Step 3
Aggregated, deduplicated datasets ready for Data Warehouse loading.

---

## STEP 4 – Data Warehouse Creation

### Purpose
The Data Warehouse stores **clean, structured data** optimized for **SQL queries and analytics**.

### Technology Choice: DuckDB

**DuckDB was chosen because:**
- **Runs locally**: No server infrastructure required
- **Requires no server**: Embedded database (single file)
- **Requires no license**: Free and open-source
- **Supports standard SQL**: Full SQL compliance (CTEs, window functions, aggregations)
- **Optimized for analytical workloads**: Columnar storage, vectorized execution

### Storage
The Data Warehouse is stored as a local file:
```
warehouse/realestate.duckdb
```

### Tables Created

#### Table 1: `monthly_indicators`
- **Source**: Curated `monthly_indicators.csv`
- **Purpose**: Time series of market metrics by geography and month
- **Key Columns**:
  - `annee_mois`: Year-month (YYYY-MM)
  - `code_geo`: Geographic code
  - `libelle_geo`: Geographic name (department)
  - `med_prix_m2_appartement`: Median apartment price (€/m²)
  - `moy_prix_m2_appartement`: Mean apartment price (€/m²)
  - `med_prix_m2_maison`: Median house price (€/m²)
  - `moy_prix_m2_maison`: Mean house price (€/m²)
  - `nb_ventes_appartement`: Apartment transaction count
  - `nb_ventes_maison`: House transaction count
- **Use Cases**: Time series analysis, year-over-year evolution, regional trends

#### Table 2: `top20_by_transactions`
- **Source**: Curated `top20_by_transactions.csv`
- **Purpose**: Rankings of most active departments by transaction volume
- **Key Columns**:
  - `code_geo`: Department code
  - `libelle_geo`: Department name
  - `transactions_total`: Total transactions (apartments + houses)
- **Use Cases**: Market activity analysis, volume-based rankings

#### Table 3: `top20_by_median_price`
- **Source**: Curated `top20_by_median_price.csv`
- **Purpose**: Rankings of most expensive departments by median price
- **Key Columns**:
  - `code_geo`: Department code
  - `libelle_geo`: Department name
  - `median_price_m2`: Median price per m² (€)
- **Use Cases**: Price-based rankings, prestige market identification

### Output of Step 4
Structured DuckDB warehouse with 3 optimized tables for analytical queries.

---

## STEP 5 – Data Warehouse Validation

### Purpose
Validate the integrity and correctness of the Data Warehouse.

### Validation Checks

#### Data Completeness
- All curated CSV files successfully loaded
- No missing required columns in DuckDB tables
- Row count matches source data

#### Data Quality
- No NULL values in key columns (code_geo, annee_mois)
- Price values within reasonable range (€0 - €20,000/m²)
- Transaction counts are non-negative integers

#### Geographic Consistency
- All geographic codes resolve to valid names
- No duplicate entries for (code_geo, annee_mois) combinations
- Top 20 rankings contain exactly 20 unique departments

#### Data Integrity
- Aggregations are mathematically sound
- Price ordering is correct (top 20 by price properly sorted)
- Time series is chronologically ordered

### Validation Output
Script `05_check_warehouse.py` generates validation report confirming:
- ✅ All tables loaded successfully
- ✅ Row counts verified
- ✅ Data type consistency
- ✅ No unexpected NULL values

---

## STEP 6 – Business Intelligence Analysis

### Purpose
This layer enables **data analysis and business insights** through SQL queries on the Data Warehouse.

### Capabilities
- **SQL queries**: Complex aggregations and filtering
- **Aggregations**: Group by time, geography, property type
- **Metrics and indicators**: Medians, means, totals, growth rates
- **Export of results**: Results to CSV for reports and dashboards

### Example Analyses

#### Analysis 1: Market Trend
- **Question**: Is the real estate market increasing, decreasing, or stable?
- **Query**: Year-over-year median price evolution (apartments vs. houses)
- **Output**: `price_evolution_yoy.csv`

#### Analysis 2: Geographic Activity
- **Question**: Which departments have the most transactions?
- **Query**: Top 10 departments by transaction volume
- **Output**: `top10_departments_by_transactions.csv`

#### Analysis 3: Price Rankings
- **Question**: Which departments are most expensive?
- **Query**: Top 10 departments by median €/m²
- **Output**: `top10_departments_by_price.csv`

#### Analysis 4: Market Snapshot
- **Question**: What is the current market status?
- **Query**: Latest month indicators across all departments
- **Output**: `latest_month_2025-06.csv`

#### Analysis 5: Time Series Export
- **Question**: What are the complete historical trends?
- **Query**: Full dataset for custom analysis and visualization
- **Output**: `monthly_indicators_export.csv`

### SQL Query Examples

**Query 1: Market Trend**
```sql
SELECT annee_mois, 
       ROUND(MEDIAN(med_prix_m2_appartement), 2) as median_apt,
       ROUND(MEDIAN(med_prix_m2_maison), 2) as median_house
FROM monthly_indicators
GROUP BY annee_mois
ORDER BY annee_mois;
```

**Query 2: Top 10 Most Active Departments**
```sql
SELECT code_geo, libelle_geo, 
       SUM(nb_ventes_appartement + nb_ventes_maison) as total_transactions
FROM monthly_indicators
GROUP BY code_geo, libelle_geo
ORDER BY total_transactions DESC
LIMIT 10;
```

**Query 3: Most Expensive Departments**
```sql
SELECT code_geo, libelle_geo, 
       ROUND(MEDIAN(med_prix_m2_appartement), 2) as median_price
FROM monthly_indicators
GROUP BY code_geo, libelle_geo
ORDER BY median_price DESC
LIMIT 10;
```

### Output of Step 6
Five CSV files in `outputs/` directory ready for reporting, visualization, and stakeholder communication.

---

## OUTPUTS

### Purpose
Final results are exported as CSV files for reporting, visualization, and external consumption.

### Output Files

| File | Content | Use Case |
|------|---------|----------|
| `monthly_indicators_export.csv` | Complete time series (60+ months, all departments) | Historical analysis, trend visualization |
| `top10_departments_by_transactions.csv` | Top 10 most active departments | Market activity report |
| `top10_departments_by_price.csv` | Top 10 most expensive departments | Prestige market report |
| `price_evolution_yoy.csv` | Year-over-year price evolution (apartments & houses) | Market trend analysis, executive summary |
| `latest_month_2025-06.csv` | Snapshot of most recent month | Current market status |

### Output Location
```
outputs/
```

These files can be used for dashboards, reports, presentations, or further analysis.

---

## Data Processing Scripts

### Overview
All processing logic follows a clear sequence aligned with the 6 steps of the lab.

### Script Summary

| Step | Script | Purpose | Input | Output |
|------|--------|---------|-------|--------|
| 1 | `01_explore_raw.py` | Explore raw data structure | RAW zone CSVs | Console statistics |
| 2 | `02_clean_staging.py` | Apply cleaning rules (na→0, imputation, name filling) | RAW zone CSVs | STAGING zone CSVs |
| 3 | `03_curated_bi.py` | Geographic deduplication, department-level aggregation | STAGING CSVs | CURATED CSVs |
| 4 | `04_warehouse_duckdb.py` | Load curated data into DuckDB tables | CURATED CSVs | `realestate.duckdb` |
| 5 | `05_check_warehouse.py` | Validate warehouse integrity and data quality | DuckDB tables | Validation report |
| 6 | `06_bi_queries_duckdb.py` | Execute 6 analytical SQL queries | DuckDB tables | Console output |
| 6 | `07_export_results.py` | Export query results to CSV files | DuckDB tables | 5 CSV files |

### Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Processing** | Python 3.14 + Pandas | Data transformation and cleaning |
| **Data Lake** | Local file system + CSV | Raw and processed data storage |
| **Warehouse** | DuckDB | SQL-based analytics |
| **Scripting** | Python | Automation and orchestration |
| **Output** | CSV files | External reporting and visualization |

---

## Architecture Data Flow Summary

The data flow follows this complete logic:

```
┌────────────────────────────────────────────────────────────┐
│ STEP 1: DATA INGESTION (RAW)                              │
│ Source: Open DVF datasets (stats_dvf.csv, stats_whole.csv) │
│ Location: data_lake/raw/                                   │
│ Action: No transformation - source of truth                │
└────────────────┬───────────────────────────────────────────┘
                 ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 2: DATA CLEANING (STAGING)                           │
│ Script: 02_clean_staging.py                                │
│ Rules:                                                      │
│ • nb_ventes_* : NA → 0                                     │
│ • Prices: if count==0 → NA; if count>0 & price NA → impute│
│ • Geo names: NA → 'Inconnu'                                │
│ • Drop columns with >80% missing                           │
│ Location: data_lake/staging/                               │
│ Output: clean_monthly.csv, clean_aggregated.csv           │
└────────────────┬───────────────────────────────────────────┘
                 ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 3: BI PREPARATION (CURATED)                          │
│ Script: 03_curated_bi.py                                   │
│ Actions:                                                    │
│ • Remove EPCI if child communes exist (deduplication)      │
│ • Filter to department level (no communes/sections)        │
│ • Aggregate by month and geography                         │
│ • Remove imputation flags                                  │
│ Location: data_lake/curated/                               │
│ Output: monthly_indicators.csv, top20_* tables             │
└────────────────┬───────────────────────────────────────────┘
                 ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 4: WAREHOUSE CREATION                                │
│ Script: 04_warehouse_duckdb.py                             │
│ Technology: DuckDB (local, no server, free, SQL-optimized) │
│ Location: warehouse/realestate.duckdb                      │
│ Tables:                                                     │
│ • monthly_indicators (time series)                         │
│ • top20_by_transactions (volume rankings)                  │
│ • top20_by_median_price (price rankings)                   │
└────────────────┬───────────────────────────────────────────┘
                 ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 5: WAREHOUSE VALIDATION                              │
│ Script: 05_check_warehouse.py                              │
│ Validation: Row counts, NULL values, data types            │
│ Output: Validation report (✅ all checks passed)           │
└────────────────┬───────────────────────────────────────────┘
                 ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 6: BUSINESS INTELLIGENCE ANALYSIS                    │
│ Scripts: 06_bi_queries_duckdb.py + 07_export_results.py   │
│ Queries:                                                    │
│ • Market trend (price evolution)                           │
│ • Top 10 by transactions                                   │
│ • Top 10 by price                                          │
│ • Regional disparities                                     │
│ • Geographic correlation analysis                          │
│ Output: 5 CSV files in outputs/                            │
│                                                             │
│ Results Ready for:                                         │
│ ✓ Executive dashboards                                     │
│ ✓ Market reports                                           │
│ ✓ Stakeholder presentations                                │
│ ✓ Further statistical analysis                             │
└────────────────────────────────────────────────────────────┘
```

---
