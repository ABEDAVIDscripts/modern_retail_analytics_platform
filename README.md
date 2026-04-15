# Modern Retail Analytics Engineering Platform (BigQuery + dbt + Power BI)
<img height="400" alt="Lineage DAG updated 0" src="https://github.com/user-attachments/assets/5872dfc0-1e37-4f8f-a745-cfd8a0987d37" />

<br>

### Overview

This repository represents an end-to-end analytics engineering implementation for a retail business, covering the full lifecycle from raw data ingestion to business intelligence delivery. <br>

This implementation demonstrates how raw transactional data can be transformed into reliable, analytics-ready datasets using modern data stack tools, and ultimately consumed through interactive dashboards for decision-making.   <br>

The pipeline was designed to be efficient, scalable, and cost-optimized within BigQuery, leveraging dbt for modular, reusable transformations.

<BR>


### Business Objectives

The primary goals of this implementation:
- Establish a scalable cloud data warehouse
- Transform raw operational data into clean, structured datasets using dbt
- Implement robust data quality validation
- Design analytics-ready data marts
- Enable business insights through Power BI dashboards

<BR>
<BR>
<BR>


## Architecture & Technology

A layered ELT architecture was implemented:

```text
Raw Data (BigQuery - retail_raw)
        ↓
Staging Layer (dbt cleaning & standardization)
        ↓
Core Models (Fact + Dimensions)
        ↓
Analytics Marts (Business Aggregations)
        ↓
Power BI Dashboard
```

<BR>


### Technology Stack
| Layer          | Technology        |
| :------------- | :---------------- |
| Data Warehouse | BigQuery          |
| Transformation | dbt               |
| Data Modeling  | SQL (Star Schema) |
| Data Quality   | dbt tests         |
| Documentation  | dbt docs          |
| Visualization  | Power BI          |


<br>
<br>
<br>



## Data Warehouse (BigQuery)
<img height="400" alt="data warehouse env" src="https://github.com/user-attachments/assets/0d8ca87e-f075-489c-803e-c3dc8b226a26" /> <br>

<br>

A cloud-native data warehouse was implemented using Google BigQuery to support scalable, high-performance analytics. The environment was structured into two logical datasets to separate raw ingestion from transformed analytics layers:


### 1.  Raw Layer (`retail_raw`) 

This dataset stores the original source data exactly as ingested from operational systems. <BR>
Tables include: transactions (6,416,827), customers, products, stores, employees & discounts <BR>

Key Characteristics: <BR>
- Serves as the single source of truth
- No transformations applied (read-only layer)
- Enables traceability and reproducibility of all downstream models
- Configured with source freshness checks in dbt to simulate production monitoring

<BR>

### 2. Analytics Layer (`retail_dw`) <BR>
This dataset contains all transformed, analytics-ready models built using dbt. It is organized into structured layers:

**i. Staging Tables** <br>
- Transformation tables: stg_transactions, stg_customers, stg_products, stg_stores, stg_employees & stg_discounts


**ii. Core Data Tables (Star Schema)** <BR>
- Fact Table: `fct_transactions` (line-level transactional grain)
  - Final processed records: ~6.4M rows (6,408,257)
  - Deduplication removed ~8,500 duplicate records, ensuring a clean transactional grain
- Dimension Tables: dim_customers, dim_products, dim_stores & dim_employees.

<BR> 

**iii. Analytics Tables** <br>
- Sales: sales_daily, sales_by_store & sales_by_product <br>
  - sales_daily <br>
  <img height="350" alt="sales_daily query" src="https://github.com/user-attachments/assets/d41c8599-f4ac-4bd1-9436-462500c56b3d" /> <br>

<br>

- Customer: customer_lifetime_value & customer_rfm <br>
  - customer_rfm <br>
  <img height="300" alt="customer_rfm dw 2" src="https://github.com/user-attachments/assets/c888efcc-ec40-4fd1-bb7f-4af9b983e4f1" /> <BR>


<BR>

### Implementation Highlights
- Designed a scalable BigQuery warehouse using dataset separation (retail_raw vs retail_dw)
- Optimized storage and query performance using partitioning and clustering strategies
- Materialized analytics tables to reduce query cost and improve BI performance
- Leveraged BigQuery’s distributed processing for large-scale aggregations
- Enabled seamless integration with dbt for transformation orchestration and model management

<BR>
<br>
<br>


## Model Structure

```text
models/
│
├── staging/
│   ├── stg_transactions.sql
│   ├── stg_customers.sql
│   ├── stg_products.sql
│   └── stg_stores.sql
│   ├── stg_employees.sql
│   └── stg_discount.sql

│
├── marts/
│   ├── core/
│   │   ├── dim_customers.sql
│   │   ├── dim_products.sql
│   │   ├── dim_stores.sql
│   │   ├── dim_employees.sql
│   │   └── fct_transactions.sql
│   │
│   ├── sales/
│   │   ├── sales_daily.sql
│   │   ├── sales_by_store.sql
│   │   └── sales_by_product.sql
│   │
│   └── customer/
│       ├── customer_lifetime_value.sql
│       └── customer_rfm.sql
```

<BR>
<BR>
<BR>


## Data Transformation Process (dbt)

### 1. Staging Layer (Data Cleaning & Standardization)
Raw data was transformed to improve data quality, consistency, and usability for downstream modeling:
- Data type casting
- Column renaming and formatting
- String standardization (trim, lower)
- Deduplication using window functions:

```sql
row_number() over (
  partition by Invoice ID, Line, Product ID, SKU
  order by Invoice Total desc
)
```


✔ Established a consistent transaction grain through deduplication and standardization  <br>
✔ This layer ensures standardized, analysis-ready inputs for downstream models <br>

<br>

- Example Staging Models:

  * stg_transactions (core transformation logic) <br>
    <img height="350" alt="stg_transactions cleaning code upgrade" src="https://github.com/user-attachments/assets/09268317-e843-4abc-bd07-b15693c30513" /> <BR>


  * stg_customers (1 of 6 staging models) <br>
    <img height="350" alt="stg_customers cleaning code upgrade" src="https://github.com/user-attachments/assets/4f8bed3d-cd63-4d17-adde-fe658edcd9b0" />

<BR>
<BR>

- Sources Configuration (dbt)
```sql
version: 2

sources:
  - name: retail_raw
    database: modern-global-retail-analytics
    schema: retail_raw

    tables:

      - name: transactions
        config:
          # Freshness check included to demonstrate production monitoring
          # Dataset is static, so thresholds are intentionally large
          loaded_at_field: Date
          freshness:
            warn_after: {count: 3650, period: day}
            error_after: {count: 7300, period: day}

      - name: customers
      - name: products
      - name: stores
      - name: employees
      - name: discounts
```


<br>
<br>

### 2. Core Data Modeling

**i. Fact Model: `fct_transactions`**  <BR>
```SQL
select

    -- surrogate key for the fact table
    {{ dbt_utils.generate_surrogate_key([
      't.invoice_id',
      't.line_number',
      't.product_id',
      't.sku'
    ]) }} as transaction_key,

    -- transaction identifiers
    t.invoice_id,
    t.line_number,
    t.transaction_date,

    -- dimension foreign keys
    c.customer_key,
    p.product_key,
    s.store_key,
    e.employee_key,

    -- product attributes
    t.sku,
    t.size,
    t.color,

    -- measures
    t.quantity,
    t.unit_price,
    t.discount,
    t.line_total,
    t.invoice_total,

    -- descriptors
    t.transaction_type,
    t.payment_method,
    t.currency

from {{ ref('stg_transactions') }} t

left join {{ ref('dim_customers') }} c
    on t.customer_id = c.customer_id

left join {{ ref('dim_products') }} p
    on t.product_id = p.product_id

left join {{ ref('dim_stores') }} s
    on t.store_id = s.store_id

left join {{ ref('dim_employees') }} e
    on t.employee_id = e.employee_id
```    
 
✔ Line-level transactional data with a surrogate key generated from invoice_id, line_number, product_id, and sku. <br>
✔ Enforces uniqueness and preserves transaction-level grain through surrogate keys.

<br>

**ii. Dimension Models** <br>
 dim_customers (1 of 4 Dimension Models) <BR>
  <img height= "350" alt="dim_customers" src="https://github.com/user-attachments/assets/1f22a6ad-751b-43d3-84c9-7bbc08f90ce0" />

<br>

**iii. Data Model Schema** <br>

```sql
version: 2

models:

  - name: dim_customers
    description: Dimension table containing customer information.
    columns:
      - name: customer_key
        description: Surrogate key for customers.
        tests:
          - not_null
          - unique

      - name: customer_id
        description: Natural customer identifier from source system.
        tests:
          - not_null


  - name: dim_products
    description: Dimension table containing product details.
    columns:
      - name: product_key
        tests:
          - not_null
          - unique

      - name: product_id
        tests:
          - not_null


  - name: dim_stores
    description: Dimension table containing store information.
    columns:
      - name: store_key
        tests:
          - not_null
          - unique

      - name: store_id
        tests:
          - not_null


  - name: dim_employees
    description: Dimension table containing employee information.
    columns:
      - name: employee_key
        tests:
          - not_null
          - unique

      - name: employee_id
        tests:
          - not_null


  - name: fct_transactions
    description: Fact table containing retail transaction records.

    columns:

      - name: transaction_key
        description: Unique surrogate key for each transaction record.
        tests:
          - not_null
          - unique

      - name: customer_key
        description: Foreign key to dim_customers.
        tests:
          - relationships:
              arguments:
                to: ref('dim_customers')
                field: customer_key

      - name: product_key
        description: Foreign key to dim_products.
        tests:
          - relationships:
              arguments:
                to: ref('dim_products')
                field: product_key

      - name: store_key
        description: Foreign key to dim_stores.
        tests:
          - relationships:
              arguments:
                to: ref('dim_stores')
                field: store_key

      - name: employee_key
        description: Foreign key to dim_employees.
        tests:
          - relationships:
              arguments:
                to: ref('dim_employees')
                field: employee_key
```            
  
✔ Built using surrogate keys, optimized joins, and conformed dimensions

<br>
<br>

### 3. Data Quality Testing
Data quality validation was implemented using dbt schema.yml files, where tests were defined at the model and column level. <Br>

The following built-in dbt tests were applied:
- not_null → ensures critical fields contain no missing values
- unique → guarantees primary key uniqueness and correct data grain
- relationships → enforces referential integrity between fact and dimension tables

<br>

✔ These tests ensured:
- Primary key integrity (e.g., transaction_key, customer_key are unique and non-null)
- Referential integrity (all foreign keys in fact tables correctly map to dimension tables)
- Reliable joins across models (no orphaned or inconsistent records) <br>

> All tests were declaratively defined in schema.yml, following production-grade best practices.


<br>
<br>

### 4. Data Freshness & Monitoring
Source freshness checks were configured to simulate production monitoring:

```yaml
freshness:
  warn_after: {count: 3650, period: day}
  error_after: {count: 7300, period: day}
```

<br>
<br>

### 5. dbt Execution

Models were built and validated using dbt commands:

```bash
dbt run
dbt test
```

<br>

- dbt Run Result <br>
<img height="200" alt="dbt run recent" src="https://github.com/user-attachments/assets/1f690c3b-a111-4e77-bf55-88b4082ee753" /> <br>

<br>


- dbt Test Result <br>
<img height="200" alt="dbt test recent" src="https://github.com/user-attachments/assets/38908749-bd9e-4d48-8516-9dda8070d9c0" /> <br>

<br>
<br>


### 6. Analytics Marts
**i. Sales Analytics** <br>
- sales_daily:  <br>
  <img  height="350" alt="sales_daily" src="https://github.com/user-attachments/assets/abf506c5-a826-4ea1-93d9-a9ecbd63d535" /> <br>

<br>

- sales_by_store: <br>
  <img height="350" alt="sales_by_store" src="https://github.com/user-attachments/assets/e8cb3be1-c154-41e6-8ba4-b39ef27db289" /> <br>

<br>

- sales_by_product: <br>
  <img height="350" alt="sales_by_product" src="https://github.com/user-attachments/assets/fb5a50ca-8c92-4e65-bc7a-7ade0fe7823a" />

<br>  
<br>

**ii. Customer Analytics** <br>

- customer_lifetime_value
```sql
{{ config(materialized='table') }}

with customer_base as 
(
    select
      c.customer_id, c.customer_name, c.city, c.country,
      f.store_key,
      count(*) as transactions

    from {{ ref('fct_transactions') }} f

    left join {{ ref('dim_customers') }} c
      on f.customer_key = c.customer_key

    group by
      c.customer_id, c.customer_name, c.city, c.country,
      f.store_key
),

ranked_store as 
(
    select *,
      row_number() over(partition by customer_id order by transactions desc) as rn
    from customer_base
)

select
    c.customer_id, c.customer_name, c.city, c.country,
    s.store_name as primary_store,

    count(distinct f.invoice_id) as total_orders,
    sum(f.line_total) as lifetime_revenue,
    avg(f.line_total) as avg_order_value,

    min(f.transaction_date) as first_purchase_date,
    max(f.transaction_date) as last_purchase_date,

    date_diff(max(f.transaction_date), min(f.transaction_date), day) as customer_tenure_days

from {{ ref('fct_transactions') }} f

left join {{ ref('dim_customers') }} c
    on f.customer_key = c.customer_key

left join ranked_store rs
    on c.customer_id = rs.customer_id and rs.rn = 1

left join {{ ref('dim_stores') }} s
    on rs.store_key = s.store_key

group by
    c.customer_id,
    c.customer_name,
    c.city,
    c.country,
    s.store_name
```    

<br>

- customer_rfm
<img height="350" alt="customer_rfm" src="https://github.com/user-attachments/assets/2b30d85d-2143-409d-aa3d-b9bfcf0d99ec" />

<br>

✔ Enables customer segmentation and behavioral analysis

<br>
<br>


### 7. Data Lineage (dbt DAG)
The lineage graph illustrates the full dependency flow from raw data sources to final analytics marts. <br>
This ensures transparency, traceability, and impact analysis across the pipeline. <br>

<img height="400" alt="Lineage DAG updated 0" src="https://github.com/user-attachments/assets/e9d69f04-edc6-406b-b37e-ed2693010754" /> <br>

<img height="400" alt="customer_rfm doc 2" src="https://github.com/user-attachments/assets/8f4c9025-bcff-45ab-9d61-f5232bd915f2" />

<br>
<br>

### 8. Performance Optimization
- Aggregation tables (marts) reduce query cost in BigQuery
- Clustered tables improve query performance
- Partitioned large tables (transactions) by transaction_date to reduce query scan costs
- Applied clustering on frequently filtered columns (e.g., store_key, product_key) to improve query performance

<br>
<br>
<br>

## Business Intelligence (Data Visualization)
A multi-page dashboard was developed in Power BI to support business decision-making:

#### 1. Executive Overview <br>
<img height="300" alt="executive overview" src="https://github.com/user-attachments/assets/1c353651-ac27-461f-b200-cd628cbf530b" /> <br>

  - Total Revenue, Orders, Discount and Gross Sales KPI
  - Revenue & order trends over time

<br>


#### 2. Store Performance <br>
<img height="300" alt="store performance" src="https://github.com/user-attachments/assets/a490e43c-ce98-46d5-b17b-26e280f9fa81" /> <br>

  - Revenue by store
  - Order distribution
  - Store records

<br>


#### 3. Product Performance <br>
<img height="300" alt="product performance" src="https://github.com/user-attachments/assets/4d293d1f-d796-40cf-bed9-99be5b335375" /> <br>

  - Category Performance
  - Top-performing products
  - Revenue vs Unit sold

<br>

#### 4. Customer Insights <br>
<img height="300" alt="customer insight" src="https://github.com/user-attachments/assets/4a9f5d4a-3207-4999-b4bc-7e848d1da3d3" /> <br>

  - Customer records
  - Customer segmentation (RFM)
  - Purchase distribution


<br>
<br>

### Key Insights

- Executive Overview
  - Total revenue: $733.09M across the full period
  - 2024 was the peak year at $383.1M in revenue and 2.23M orders
  - Total discount rate is ~0.1%; pricing discipline is strong
  - December shows a massive seasonal spike (~$127M revenue, ~1M orders)

- Product Performance
  - Coats & Blazers ($100M) and Pants & Jeans ($94M) are the top two products, making up ~27% of total revenue.
  - Top 5 products (adding Suits & Blazers, Sportswear, Suits & Sets) account for the majority of revenue.
  - Children's category ($0.07bn) is ~5x smaller than Feminine ($0.34bn) and Masculine ($0.33bn); a largely untapped segment.
  - A few premium products generate high revenue at low volume, these lines need protection.

- Store Performance
  - Chinese stores collectively dominate at $69M–$132M per store, with Shanghai leading at $132M.
  - US stores (New York $21M, LA $20M, Houston $14M, Chicago $11M, Phoenix $9M) average around ~$15M; significantly underperforming vs Asia
  - European stores trail further, with Berlin the strongest at $11M and Madrid, Lisboa, and Paris clustered at $8M each.
  - Heavy geographic concentration in China creates a single-region dependency risk

- Customer Insights
  - Over 50% of customers are recent low-value buyers, indicating strong acquisition but weak monetization.
  - The majority of customers make only 1–3 purchases, showing low retention and limited repeat behavior.
  - A small segment of customers generates high order volume and revenue, highlighting the importance of loyal/VIP customers.
  - High-value customers are a minority, meaning revenue is concentrated among a few top customers.
  - Customer activity is concentrated in major cities (e.g., Shenzhen, Shanghai, Guangzhou), suggesting strong urban market dependence.

<br>

### Key Engineering Decisions

- Adopted a star schema for analytical performance and simplicity
- Used surrogate keys to maintain consistency across joins
- Built modular dbt models to ensure reusability and scalability
- Created aggregated marts to reduce query cost in BI layer

<br>

### Scalability Considerations

- Data models designed to support incremental loading for efficient future data ingestion
- BigQuery used for distributed query processing at scale
- dbt enables modular expansion of new business domains


<br>
<br>
