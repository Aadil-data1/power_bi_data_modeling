# Power BI End-to-End Data Modeling Portfolio Project: Data Model Transformation

A comprehensive end-to-end data modeling portfolio project transforming a chaotic, unoptimized  data set into a production-ready, highly performant Star Schema data architecture in Power BI.

---

## Executive Summary & Problem Statement

In enterprise Business Intelligence environments, report performance degradation and inaccurate metrics are rarely caused by dashboard visual design or DAX complexity alone. In most instances, the root cause lies in an underlying flawed data model.

When source systems or disparate tables are directly loaded into Power BI without proper data modeling, organizations suffer from severe performance bottlenecks, circular filter paths, ambiguous many-to-many relationships, heavy file sizes, and inconsistent reporting metrics.

### The Data Model (Baseline State)

The initial baseline data set represents a typical unmodeled enterprise scenario:

* **23 Raw Tables** loaded directly from transactional systems and staging sources without unified modeling.
* **Many-to-Many Relationships & Bi-directional Filters** spanning unstructured transactional tables.
* **Inconsistent Grain & Duplicate Attributes**: Transactional header and detail lines mixed together, causing data fanning out and inflated metric totals.
* **Non-Standardized Conventions**: Mixed naming conventions (PascalCase, camelCase, technical system names), inconsistent data types, string hash keys, redundant columns, and mixed languages.
* **Data Quality Anomalies**: Duplicate entity records across source systems, dummy test records (e.g., test customer ID `999`), missing primary keys, and comma-separated lists stored within text fields.

### The Solution: Star Schema Architecture

This project systematically refactors the chaotic dataset into a clean, enterprise-grade Star/Galaxy Schema architecture. By establishing explicit single-direction `1-to-Many` relationships from shared dimensions to fact tables, the transformed data model ensures protected metric totals, optimal Power BI engine compression, and seamless cross-filtering.

---

## Core Data Modeling Governance Rules

Throughout the transformation process, five mandatory data modeling governance rules were enforced:

1. **Star Schema Enforcement**: Fact tables are placed in the center surrounded by dimension tables. Fact tables are never connected directly to other fact tables; all cross-fact analysis occurs through shared dimensions. Filter directions strictly flow single-direction from dimensions (`1`) to facts (`Many`).
2. **Grain Verification ("Say it Out Loud")**: The grain of every table is explicitly defined and validated prior to performing merges or relationships to prevent duplicate rows and metric distortion.
3. **Column Earns Its Place**: Unnecessary columns—such as source system string hash keys, legacy system references, internal audit flags, and long free-text notes—are aggressively removed to maximize Power BI columnar compression and eliminate visual canvas clutter.
4. **Protection of Core Metrics**: Baseline financial and operational numbers (e.g., total sales line amount) are monitored before and after every query merge and data transformation to guarantee mathematical accuracy.
5. **Universal Naming & Structural Standards**:  
  * **Language**: English across all entities and attributes.
  * **Naming Case**: Unified `snake_case` naming convention for all table and column names.
  * **Table Prefixes**: `dim_` for dimension tables, `fact_` for fact tables.
  * **Key Suffixes**: Surrogate keys generated within the data model end with `_key` (e.g., `customer_key`, `product_key`), whereas business keys originating from source systems retain `_id` (e.g., `customer_id`).
  * **Attribute Formatting**: Proper capitalization applied to display attributes, and standardized `yyyy-MM-dd` date formatting applied across all temporal fields.

---

## Workspace Architecture in Power Query

To maintain clear separation between raw staging data, intermediate transformations, and final data model entities, the Power Query ETL pipeline is organized into five structured query groups:

```
Power Query Query Groups
├── 1. Stage          (23 Raw source queries preserved in original state)
├── 2. Dimensions     (Consolidated, cleaned dimension tables)
├── 3. Facts          (Refactored fact tables at aligned granularities)
├── 4. Support        (Auxiliary tables, e.g., security mapping tables)
└── Other Queries     (Inbox for newly imported sources prior to triage)

```

---

## Phase-by-Phase Implementation Workflow

### Phase 1: Exploration &amp; Preparation

1. **Entity & Grain Audit**: Systematically inspected all 23 raw tables in Power BI Table View to categorize candidate dimensions (descriptive context) and candidate facts (event transactions and numeric metrics).
2. **Power Query Staging Setup**: Isolated raw queries into the `1. Stage` folder and disabled direct loading into the report model where applicable to prevent visual model bloat.

### Phase 2: Dimension Engineering &amp; Standardization

Multiple source tables representing identical real-world entities were merged, cleansed, and standardized into consolidated dimension tables:

* **`dim_customer`**:  
  * Consolidated `customer_master`, `customer_contacts`, `user_details`, `addresses`, `cities`, and `regions`.
  * Filtered out test customer ID `999`.
  * Resolved grain misalignment with contact lists by filtering `customer_contacts` strictly to primary contacts (`is_primary = true`), preserving a strict `1-to-1` relationship prior to merging.
  * Resolved entity terminology ambiguity between "customer" and "user" by standardizing on `customer`.
  * Removed unneeded audit attributes including `hash_keys`, `source_id`, `account_manager`, and redundant address IDs.
* **`dim_product`**:  
  * Derived from raw `products` and `subcategories` tables.
  * Extracted pipe-delimited categories and subcategories (`Category | Subcategory`) using delimiter splitting and applied word capitalization.
  * Resolved data quality anomalies where duplicate records existed for products (e.g., duplicate rows for kitchen and audio items lacking attribute completeness) by filtering bad source rows via `source_id`.
  * Generated an explicit surrogate key `product_key` using index column generation.
* **`dim_geo`**:  
  * Constructed a dedicated geography dimension from the raw `cities` table containing `city` and `region` attributes.
  * Generated a surrogate `geo_key` to serve as a role-playing dimension for both shipping locations (`ship_to_city_key`) and billing locations (`bill_to_city_key`).
* **`dim_campaign`**:  
  * Isolated descriptive campaign context (campaign name, channel, budget, start date, end date) from daily event logs.
  * Assigned surrogate `campaign_key`.
* **`dim_order_flags` (Junk Dimension)**:  
  * Combined discrete transactional attributes—`order_channel`, `status`, and `priority`—from order headers into a single junk dimension table to eliminate high-cardinality flags from fact tables.
  * Enriched numerical channel codes (`10`, `20`, `30`, `40`) by creating a manual lookup mapping table (`channels`) translating codes into business-friendly terms: `Online Store`, `Retail Partner`, `Wholesale`, and `Field Sale`.
* **`dim_date` (Shared Calendar Dimension)**:  
  * Programmatically generated an automated calendar table in DAX using `CALENDARAUTO()` to dynamically encompass the full date range (`2025-01-01` through `2026-12-31`).
  * Extracted structural temporal attributes including `year` and `month`.
  * Formatted all date outputs to `yyyy-MM-dd`.

### Phase 3: Fact Modeling &amp; Grain Realization

Handling header and detail transactions is one of the most critical aspects of analytical data modeling. Creating separate fact tables for headers and details and attempting to join them directly leads to incorrect metrics and poor query execution.

**The Golden Pattern**: Core fact tables must be modeled at the lowest, finest detail level (item line level), referencing header attributes via lookups.

* **`fact_sales`**:  
  * Appended historical split tables `orders_2025` and `orders_2026` into a unified `orders` header table, dropping legacy columns and free-text order notes.
  * Modeled core facts from `order_line_items` (finest grain).
  * Performed lookups against dimension tables to replace natural keys and names with integer surrogate keys (`customer_key`, `product_key`, `flag_key`, `ship_to_city_key`, `bill_to_city_key`).
  * Dropped pre-aggregated header total metrics (`order_total`) to prevent reliance on stale pre-calculated values; line items are dynamically aggregated in DAX.
  * Removed redundant dimension attributes (`customer_city`, `region_name`) already hosted in `dim_customer`.
* **`fact_inventory`**:  
  * Transformed wide monthly snapshot columns (`Jan`, `Feb`, ..., `Dec`) into a tall, normalized fact structure using Power Query's `Unpivot Columns` transformation.
  * Converted text month descriptors into formal dates and mapped `product_key`.
* **`fact_campaign_spend`**:  
  * Isolated daily performance metrics (`impressions`, `clicks`, `spend`) into a daily campaign transactional fact table linked via `campaign_key`.
* **`fact_sales_targets`**:  
  * Normalized monthly revenue targets into a period-based target fact table for actual-versus-target variance reporting.

### Phase 4: Advanced Modeling Patterns

#### 1\. Accumulating Snapshot Fact Table (`fact_order_process`)

To support process efficiency analytics and end-to-end lifecycle tracking (e.g., order-to-delivery lead times and payment latency), an accumulating snapshot fact table was constructed.

* **Structure**: Each row represents a single order lifecycle instance containing sequential milestone dates:  
  * `order_date`
  * `ship_date`
  * `delivery_date`
  * `invoice_date`
  * `payment_date`
* **Implementation**: Merged milestone dates from `orders`, `shipments`, `invoices`, and `payments` using `order_id` and `invoice_id` lookups. This single table enables seamless lead time calculations without requiring complex cross-table joins across five separate transactional event tables.

#### 2\. Factless Fact Table (`fact_promotion_coverage`)

To evaluate which products were eligible or covered under marketing campaigns without introducing artificial numeric metrics:

* Exploded comma-separated product lists from `campaign_skus` into discrete rows.
* Mapped `campaign_key` and `product_key`.
* Serves as a pure relationship bridge (Factless Fact) tracking occurrence and coverage rather than numeric amounts.

---

## Refactored Data Model Architecture (star schema)

The final architecture comprises 6 clean dimension tables and 6 dedicated fact tables connected via single-direction `1-to-Many` relationships.

```
+-----------------------------------------------------------------------------------+
|                               SHARED DIMENSIONS                                   |
|  dim_customer | dim_product | dim_geo | dim_campaign | dim_order_flags | dim_date  |
+-----------------------------------------------------------------------------------+
                                         | (1)
                                         |
                                         | (Many)
+-----------------------------------------------------------------------------------+
|                                  FACT TABLES                                      |
|  1. fact_sales            (Core transactional sales &amp; line items)                  |
|  2. fact_inventory        (Monthly inventory stock tracking)                      |
|  3. fact_campaign_spend   (Daily marketing spend &amp; click performance)             |
|  4. fact_sales_targets    (Monthly revenue targets)                               |
|  5. fact_order_process    (Accumulating snapshot of order fulfillment lifecycle)   |
|  6. fact_promotion_coverage (Factless fact mapping campaign product coverage)     |
+-----------------------------------------------------------------------------------+

```

---

## Centralized DAX Measures Repository

All calculations are decoupled from physical tables and hosted within a dedicated, centralized DAX measures container table (`_measures`).

### 1\. Financial &amp; Volume Measures

```
// Total Line Revenue
total_sales =
SUM(fact_sales[line_total])

```

```
// Distinct Order Volume (Accounting for line-item grain)
total_orders =
DISTINCTCOUNT(fact_sales[order_id])

```

```
// Total Ordering Customers
total_active_customers =
DISTINCTCOUNT(fact_sales[customer_key])

```

```
// Total Master Customers
total_customers =
COUNT(dim_customer[customer_key])

```

### 2\. Process Efficiency &amp; Lifecycle Measures

```
// Order-to-Payment Latency (Calculated Column on fact_order_process)
order_to_pay_days =
DATEDIFF(
    fact_order_process[order_date],
    fact_order_process[payment_date],
    DAY
)

// Average Order-to-Payment Days Measure
average_order_to_pay =
AVERAGE(fact_order_process[order_to_pay_days])

```

---

## Security &amp; Row-Level Security (RLS) Architecture

To enforce data privacy compliance and regional data segregation:

1. **Security Mapping**: Integrated the `security` support table containing mapping records between `user_email` and authorized `region`.
2. **Relationship Path**: Established a `1-to-Many` single-direction filter flow from `security[region]` to `dim_customer[region]`. Because `dim_customer` filters `fact_sales` and `fact_order_process`, security filters automatically propagate downstream to all financial and lifecycle metrics.
3. **DAX RLS Filter Rule**: Defined the `regional_access` role on `dim_customer`:

```
// Regional Access Filter Expression on dim_customer
[region] = LOOKUPVALUE(
    security[region],
    security[user_email],
    USERPRINCIPALNAME()
)

```

Validation via Power BI "View as Roles" confirmed that authenticated users (e.g., regional managers) are strictly restricted to data points corresponding to their assigned territory, while baseline model totals remain fully protected.

---

## Key Learning Takeaways & Best Practices

1. **Invest in Upfront Exploration**: Thoroughly analyzing raw tables, relationships, and grains prior to building queries prevents circular dependencies and broken measures later in development.
2. **Standardize Query Workspace**: Organizing queries into explicit stage, dimension, fact, and support groups brings enterprise maintainability to Power BI projects.
3. **Always Protect the Numbers**: Monitoring card visuals for core metrics before and after merges guarantees that grain misalignments and row duplications are caught immediately.
4. **Prefer Line-Level Fact Grain**: Building fact tables from detail line items rather than header records provides maximum analytical drill-down capability and eliminates duplicate header aggregation errors.
5. **Eliminate Redundant Columns**: Dropping source audit hash keys and unneeded attributes optimizes columnar compression, reduces `.pbix` file size, and accelerates report rendering.

About me

Hi i am** Aadil Hameed **, completed a comprehensive Power BI Data Modeling project, eager to plunge into data field; ready to deliver the best possible results and catalyze businesses to make productive decisions with quality data analysis.
