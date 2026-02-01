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

---

## 6. Data Source Layer

### Description
The data source is open French government data from the DVF (Direction de la Validation Foncière) real estate registry.

### Data Files
- **stats_dvf_mth.csv**: Monthly aggregated statistics (July 2020 - June 2025)
  - Geographic hierarchies (communes, EPCI, departments, regions)
  - Price metrics: median and mean €/m² for apartments and houses
  - Transaction counts by property type
  
- **stats_agg.csv**: Global aggregated statistics (reference data)
  - Regional and national-level aggregations

### Characteristics
- Public and free
- Aggregated data (not individual transactions)
- Missing values for low-volume zones (privacy protection)
- 5 years of historical data (60+ monthly observations)
- Geographic hierarchy with ~36,000 locations

### Role
The data source represents the entry point of the data pipeline. It provides authoritative government data on French real estate transactions.

---

## 7. Data Lake Layer

### Purpose
The Data Lake stores data as files and preserves data flexibility and history.

### Technology
- Local file system
- CSV files
- Python for processing
- Pandas library for data manipulation

### Data Lake Zones

#### RAW Zone
- **Location**: `data_lake/raw/`
- **Contents**: `stats_dvf_mth.csv`, `stats_agg.csv`
- **Characteristics**:
  - Stores original data
  - No transformation applied
  - Acts as the source of truth
  - Never modified
- **Purpose**: Audit trail and reproducibility

#### STAGING Zone
- **Location**: `data_lake/staging/`
- **Contents**: 
  - `clean_monthly.csv` (processed monthly statistics)
  - `clean_aggregated.csv` (processed national aggregates)
- **Characteristics**:
  - Cleaned and standardized data
  - Missing values handled with hierarchical imputation
  - Geographic name normalization
  - Quality flags added (`_imputation_level` columns)
  - Duplicates removed
- **Imputation Strategy**:
  - Level 0: Real data (no imputation needed)
  - Level 1: Department median (first fallback)
  - Level 2: Region median (second fallback)
  - Level 3: Global median (last resort)
  - NA: No data available at any level
- **Purpose**: Bridge between raw data and curated analytics

#### CURATED Zone
- **Location**: `data_lake/curated/`
- **Contents**:
  - `monthly_indicators.csv` (time series by geography)
  - `top20_by_transactions.csv` (top 20 most active departments)
  - `top20_by_median_price.csv` (top 20 most expensive departments)
- **Characteristics**:
  - BI-ready data
  - Selected columns only (no imputation flags)
  - Aggregated to department level (for rankings)
  - Optimized for analytics queries
  - No missing prices (already imputed in staging)
- **Purpose**: Input for the Data Warehouse and direct BI consumption

---

## 8. Data Processing Layer

### Technology
- Python 3.14
- Pandas library
- Pathlib for file management

### Scripts and Responsibilities

| Script | Purpose | Input | Output |
|--------|---------|-------|--------|
| `01_explore_raw.py` | Explore raw data structure | `stats_dvf.csv` | Console output + statistics |
| `02_clean_staging.py` | Clean data and apply hierarchical imputation | `stats_dvf.csv`, `stats_whole.csv` | `clean_monthly.csv`, `clean_aggregated.csv` |
| `03_curated_bi.py` | Prepare BI-ready datasets, remove duplicates | Staging CSVs | Curated CSVs (monthly_indicators, top20 tables) |
| `04_warehouse_duckdb.py` | Load curated data into DuckDB | Curated CSVs | `realestate.duckdb` |
| `05_check_warehouse.py` | Validate warehouse integrity | DuckDB tables | Validation report |
| `06_bi_queries_duckdb.py` | Execute analytical SQL queries | DuckDB tables | Query results (console output) |
| `07_export_results.py` | Export results to CSV files | DuckDB tables | 5 CSV files in `outputs/` |

### Key Processing Logic

**Data Cleaning (Script 02)**:
- Remove duplicates
- Fill missing values using hierarchical strategy
- Normalize geographic names
- Track imputation quality with flags

**Deduplication (Script 03)**:
- Remove EPCI (inter-communal groupings) when child communes exist
- Prevents double-counting in hierarchical data
- Filters to department level for final rankings

**Data Preservation**:
- Processing logic is clearly separated from storage
- All transformations are documented and reproducible

---

## 9. Data Warehouse Layer

### Purpose
The Data Warehouse stores clean, structured data optimized for SQL queries and analytics.

### Technology Choice: DuckDB

DuckDB was chosen because:
- **Embedded**: Runs locally without server setup
- **No Infrastructure**: Single-file database (`.duckdb`)
- **No License**: Free and open-source
- **SQL Standard**: Full support for standard SQL (CTEs, window functions, aggregations)
- **CSV Integration**: Native `read_csv_auto()` function for seamless data import
- **Development Speed**: No schema pre-definition required

### Storage

The Data Warehouse is stored as a local file:
```
warehouse/realestate.duckdb
```

### Tables

#### Table: `monthly_indicators`
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

#### Table: `top20_by_transactions`
- **Source**: Curated `top20_by_transactions.csv`
- **Purpose**: Rankings of most active departments by transaction volume
- **Key Columns**:
  - `code_geo`: Department code
  - `libelle_geo`: Department name
  - `transactions_total`: Total transactions (apartments + houses)
- **Use Cases**: Market activity analysis, volume-based rankings

#### Table: `top20_by_median_price`
- **Source**: Curated `top20_by_median_price.csv`
- **Purpose**: Rankings of most expensive departments by median price
- **Key Columns**:
  - `code_geo`: Department code
  - `libelle_geo`: Department name
  - `median_price_m2`: Median price per m² (€)
- **Use Cases**: Price-based rankings, prestige market identification

---

## 10. Analytics and BI Layer

### Purpose
This layer enables data analysis and business insights through SQL queries.

### Capabilities
- Complex SQL queries (aggregations, filtering, grouping)
- Time series analysis (monthly trends)
- Geographic comparisons (regional disparities)
- Metrics and KPIs (medians, means, counts)
- Correlation analysis (price vs. volume)
- Export of results for external consumption

### Example Analyses

**Market Trend Analysis**:
- Year-over-year price evolution (apartments vs. houses)
- Identification of growth periods and slowdowns
- Peak pricing identification

**Geographic Analysis**:
- Most active departments by transaction volume
- Most expensive departments by median price
- Regional price disparities
- Correlation between price and activity level

**Data Quality Assessment**:
- Distribution of imputation levels
- Coverage analysis by geographic zone
- Missing data identification

### Query Examples

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
SELECT code_geo, libelle_geo, SUM(nb_ventes_appartement + nb_ventes_maison) as total_transactions
FROM monthly_indicators
GROUP BY code_geo, libelle_geo
ORDER BY total_transactions DESC
LIMIT 10;
```

**Query 3: Most Expensive Departments**
```sql
SELECT code_geo, libelle_geo, MEDIAN(med_prix_m2_appartement) as median_price
FROM monthly_indicators
GROUP BY code_geo, libelle_geo
ORDER BY median_price DESC
LIMIT 10;
```

---

## 11. Outputs

### Purpose
Final results are exported as CSV files for reporting, visualization, and external consumption.

### Output Files

| File | Content | Use Case |
|------|---------|----------|
| `monthly_indicators_export.csv` | Complete time series (60+ months, all departments) | Historical analysis, trend visualization |
| `top10_departments_by_transactions.csv` | Top 10 most active departments | Market activity report |
| `top10_departments_by_price.csv` | Top 10 most expensive departments | Prestige market report |
| `price_evolution.csv` | Year-over-year price evolution (apartments & houses) | Market trend analysis, executive summary |
| `latest_month_2025-06.csv` | Snapshot of most recent month | Current market status |

### Output Location
```
outputs/
```
These files can be used for dashboards, reports, presentations, or further analysis.

---

## 12. Data Quality Considerations

### Quality Flags
Each price column in staging data includes an `_imputation_level` flag indicating data confidence:
- **0**: Real data (no imputation)
- **1**: Department-level median (imputed)
- **2**: Region-level median (imputed)
- **3**: Global median (imputed)

### Quality Limitations
1. **Aggregation Level**: DVF provides aggregated statistics, not individual transactions
2. **Geographic Gaps**: Some cadastral sections have no transactions (prices missing)
3. **Privacy Protection**: Low-volume zones are suppressed in official data
4. **Category Limitation**: Only apartment/house distinction (no property-level attributes)
5. **Temporal Lag**: Latest months (2025-2026) may have incomplete data

### Data Quality Best Practices Implemented
- RAW data never modified (immutability)
- Quality flags track imputation decisions
- Geographic deduplication removes double-counting
- Hierarchical imputation preserves regional variation
- Version control for reproducibility

---

## 13. Architecture Flow Summary

The data flow follows this logic:

```
1. RAW LAYER
   ↓
   stats_dvf.csv, stats_whole.csv
   (Original French government data)
   
2. STAGING LAYER
   ↓
   Script 02: Clean, normalize, hierarchically impute prices
   Add quality flags (_imputation_level)
   
3. CURATED LAYER
   ↓
   Script 03: Deduplicate hierarchies, aggregate to meaningful levels
   Prepare BI-ready datasets
   
4. WAREHOUSE LAYER
   ↓
   Script 04: Load curated data into DuckDB tables
   
5. ANALYTICS LAYER
   ↓
   Script 06: Execute SQL queries for insights
   
6. OUTPUT LAYER
   ↓
   Script 07: Export results to CSV files
   (For dashboards, reports, external consumption)
```

---

## 14. Technology Stack Summary

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Data Source** | French DVF (CSV) | Real estate statistics |
| **Data Lake** | Local file system + CSV | Raw and processed data storage |
| **Processing** | Python 3.14 + Pandas | Data transformation and cleaning |
| **Data Warehouse** | DuckDB | SQL-based analytics |
| **Scripting** | Python | Automation and orchestration |
| **Output** | CSV files | External reporting and visualization |

---

## Conclusion

This architecture demonstrates modern data engineering best practices:
- **Separation of Concerns**: Clear layers with defined responsibilities
- **Data Governance**: Immutable raw data, tracked transformations
- **Quality Assurance**: Hierarchical imputation with quality flags
- **Scalability**: Open-source tools suitable for lab and production environments
- **Accessibility**: SQL-based analytics enabling diverse analytical use cases

The platform successfully ingests, transforms, and analyzes 5 years of French real estate data, providing actionable business intelligence for market analysis and decision-making.

