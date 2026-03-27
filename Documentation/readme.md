## Data Pipeline Overview

The project follows the Medallion Architecture and transforms raw operational data into analytics‑ready KPIs.  
The flow:

**Raw Files → Bronze → Silver → Gold (Facts & Dimensions) → Data Cube → KPI Views**

The system processes six operational datasets:  
store, order, estimate, invoice, customer_survey, ns_budget.

Azure Blob Storage is used as the raw data input.

<br>

## Bronze Layer – Raw Landing Zone

The Bronze layer acts as the unmodified landing zone.  
All datasets are ingested exactly as received from Azure Blob Storage:

- schema preserved as in source  
- raw timestamps retained  
- raw text formatting, casing, special characters preserved  
- raw duplicates present  
- raw Fivetran metadata columns retained  

This layer serves as a historical archive and point of auditing.

<br>

## Silver Layer – Detailed Transformations  
**Notebook:** `02_silver/nb_silver`

This notebook performs complete cleaning, standardization, and validation of all Bronze tables.

#### General Operations Included
- Standardizes all column names using a custom function (`normalize_columns`)  
- Removes metadata fields such as `_file`, `_line`, `_fivetran_synced`, `_modified`  
- Removes duplicates across all tables  
- Normalizes text fields (lowercase, trim spaces)  
- Converts all IDs, numeric fields, and amounts to proper numeric types  
- Converts timestamps to timestamp type and derives date fields  
- Ensures referential consistency using cross‑table joins  

#### Store Table Transformations
- Drops all ingestion metadata columns  
- Converts `store_id` and `opened_year` to integers  
- Fixes missing `manager_name` values using a mapping join to a clean source  
- Forces consistent formatting for:
  - city  
  - state  
  - store_type  
- Removes duplicate store rows  
- Ensures uniform data types for store attributes  

#### Order Table Transformations
- Removes ingestion metadata and duplicates  
- Converts `store_id` to integer  
- Standardizes:
  - service_type  
  - vehicle_make  
  - vehicle_model  
  - order_status  
- Adds derived lifecycle metrics:
  - `days_in_shop`  
  - `vehicle_in_to_work_start_days`  
  - `work_start_to_completion_days`  
  - `work_completion_to_delivery_days`  
  - `planned_vs_actual_completion_days`  
  - `promised_vs_actual_delivery_days`  

These derived fields enable cycle‑time KPIs and operational stage analysis.

#### Customer Survey Table Transformations
- Converts numerical ratings to integer type  
- Converts `responded_flag` to boolean  
- Joins with orders to attach the correct `store_id`  
- Ensures survey timestamps are valid and standardized  
- Removes duplicates and metadata fields  

#### Estimate Table Transformations
- Preserves full timestamp `created_ts`  
- Adds `created_date` for modeling  
- Standardizes:
  - estimator_name  
  - estimate_type  
- Converts `estimate_amount` to double  
- Removes metadata and duplicates  

#### Invoice Table Transformations
- Converts `invoice_amount` to double  
- Extracts `invoice_date` from timestamp  
- Standardizes:
  - payment_mode to lowercase  
  - currency to uppercase  
- Removes duplicate invoice records  

#### NS Budget Table Transformations
- Converts `ns_store_id` to integer and renames it to `store_id`  
- Renames `month` to `budget_date`  
- Derives:
  - `budget_year`  
  - `budget_month`  
- Converts `budget_amount` to double  
- Removes duplicates and Fivetran metadata  

#### Silver Output Summary
All six tables are now:
clean, deduplicated, standardized, typed correctly, and enriched with business logic.

<br>

## Gold Layer – Dimensional Model  
Uses three notebooks:

- `03_gold/df_creation/01_base_tables`  
- `03_gold/df_creation/02_fact_tables`  
- `03_gold/df_creation/03_dim_tables`

<br>

### Fact Tables – Detailed Logic  
**Notebook:** `02_fact_tables`

##### fact_order_cycle  
Creates the primary order lifecycle fact using:
- join between orders and estimates  
- window functions to extract:
  - initial_estimate_amount  
  - final_estimate_amount  
  - estimator_id  
- removes redundant columns like raw technician names  
- preserves all lifecycle metrics from the Silver order table  

This table powers:  
cycle time KPIs, estimator accuracy KPIs, and operational performance metrics.

#### fact_invoice
Invoice fact table includes:
- invoice_id  
- order_id  
- store_id  
- technician_id  
- invoice_amount  
- payment_mode  
- currency  
- invoice timestamps  

The order table join ensures correct attribution of invoices to store and technician.

#### fact_survey
Survey fact table includes:
- survey_id  
- order_id  
- store_id  
- technician_id  
- all rating fields  
- boolean responded_flag  
- derived field: `days_to_respond`  

<br>

## Dimension Tables – Detailed Logic  
**Notebook:** `03_dim_tables`

#### dim_store
Derived from clean Silver store data. Contains:
- store_id  
- store_name  
- city  
- state  
- manager_id  
- manager_name  
- opened_year  
- store_type  

#### dim_technician
Extracted from order table.  
Ensures:
- unique technician_id  
- consistent technician_name  

#### dim_estimator
Extracted from estimate table.  
Ensures unique pairs of estimator_id and estimator_name.

#### dim_vehicle
Built from Silver order table.  
Fixes a major data quality issue:
- Removes 255 duplicate vehicles  
Vehicle dimension contains:
- vehicle_no  
- vehicle_make  
- vehicle_model  

#### dim_date
Generated for every date between:
- min(order.vehicle_in_date)  
- max(invoice_date)  

Includes attributes:
- year  
- month  
- day  
- quarter  
- day_of_week  
- week_of_year  
- is_weekend  
- date_key (YYYYMMDD)  

#### dim_budget
Budget dimension created from ns_budget with:
- store_id  
- budget_year  
- budget_month  
- budget_date  
- budget_amount  
- approved_by  

<br>

## Data Cube Layer – Denormalized Analytics Tables  
**Notebook:** `04_fact_dim_creation`

This notebook creates wide, joined analytics-friendly tables used for dashboards.

#### fact_order_dim
Combines:
- fact_order_cycle  
- dim_store  
- dim_technician  
- dim_estimator  
- dim_vehicle  
- dim_date  

Adds keys like:
- vehicle_in_date_key  
- delivery_date_key  

and enriched attributes for store, technician, estimator, and vehicle.

#### fact_invoice_dim
Enhances invoice facts with store, technician, and date dimension attributes.  
Adds invoice quarter and monthly breakdowns.

#### fact_survey_dim
Enriches survey facts with:
- store attributes  
- technician attributes  
- survey sent date  
- response date dimension keys  

#### SSOT (Single Source of Truth)
Combines all three dimensional facts:
- fact_order_dim  
- fact_invoice_dim  
- fact_survey_dim  

This becomes the unified table BI tools query.

<br>

## KPI Layer – Business Metrics  
**Notebook:** `03_gold/kpi_query/kpi`

This notebook produces all business KPIs as SQL views.

Includes the following KPI views:

- month‑to‑date performance  
- average days in shop  
- survey coverage  
- survey satisfaction score  
- revenue vs budget  
- top technicians by time accuracy  
- YTD revenue growth  
- stage cycle time breakdown  
- estimator accuracy  
- technician workload  

These are optimized for dashboard consumption.

<br>

## Pipeline Orchestration (YAML Workflow)

The pipeline runs in this order:

**Data Extraction → Data Transformation → Gold Fact/Dim Creation → Data Cube Creation → KPI Generation**

Each stage depends on the previous stage’s successful execution, preventing invalid data from reaching analytics.

<br>

