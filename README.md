# 🛍️ Modern Retail Analytics Engineering Platform

<br>

### Overview

This repository represents an end-to-end analytics engineering implementation for a retail business, covering the full lifecycle from raw data ingestion to business intelligence delivery. <br>

The work demonstrates how raw transactional data can be transformed into reliable, analytics-ready datasets using modern data stack tools, and ultimately consumed through interactive dashboards for decision-making.               

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

### 🏗️ Architecture
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
<BR>

### 🧰 Technology Stack
| Layer          | Technology        |
| :------------- | :---------------- |
| Data Warehouse | BigQuery          |
| Transformation | dbt               |
| Data Modeling  | SQL (Star Schema) |
| Data Quality   | dbt tests         |
| Documentation  | dbt docs          |
| Visualization  | Power BI          |


<BR>
<BR>

### 📂 Project Structure

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

<br>
<br>

### Data Transformation Process

#### 1. Staging Layer (Data Cleaning & Standardization)
Raw data was transformed to ensure consistency and reliability:
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

<br>

- Staging tables:
  * `stg_customers` <br>
    <img height="250" alt="stg_customers cleaning code upgrade" src="https://github.com/user-attachments/assets/4f8bed3d-cd63-4d17-adde-fe658edcd9b0" />


  * `stg_products` <br>
    <img height="250" alt="stg_products cleaning code upgrade" src="https://github.com/user-attachments/assets/91355ed5-bb99-4de4-8516-919026e7ea6c" />


  * `stg_stores` <br>
    <img height="250" alt="stg_stores cleaning code upgrade" src="https://github.com/user-attachments/assets/f9b7451f-f69c-4db9-b2e7-69dc84559cd2" />

  
  * `stg_employees` <br>
    <img height="250" alt="stg_employees cleaning code upgrade" src="https://github.com/user-attachments/assets/aca0fc82-2b2d-4627-aa7e-4d5cf1b97f58" />


  * `stg_discounts` <br>
    <img height="250" alt="stg_discount cleaning code upgrade" src="https://github.com/user-attachments/assets/d6b7e4bf-fe19-46f4-a940-105c8a94d100" />


  * `stg_transactions` <br>
    <img height="250" alt="stg_transactions cleaning code upgrade" src="https://github.com/user-attachments/assets/09268317-e843-4abc-bd07-b15693c30513" />

    <br>


- Sources <br>

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

✔ Ensured a correct and stable transaction grain

<br>
<br>

#### 2. Core Data Model

**i. Fact Table: `fct_transactions`**  <BR>
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
 
  * Line-level transactional data
  * Surrogate key generated using: <Br>
```sql
invoice_id + line_number + product_id + sku
```

✔ Guarantees uniqueness and consistent grain

<br>

**ii. Dimension Tables** <br>
* `dim_customers` <BR>
  <img height="250" alt="dim_customers" src="https://github.com/user-attachments/assets/1f22a6ad-751b-43d3-84c9-7bbc08f90ce0" />

* `dim_products` <BR>
  <img height="250" alt="dim_products" src="https://github.com/user-attachments/assets/861349c7-91da-4f2a-88b2-48ee6916b2ba" />

* `dim_stores` <BR>
  <img height="250" alt="dim_stores" src="https://github.com/user-attachments/assets/cf07a281-b2fb-4c62-b050-633fbdca22b3" />

* `dim_employees` <BR>
  <img height="250" alt="dim_employees" src="https://github.com/user-attachments/assets/b762cff8-0581-4451-85a4-5945cacf9419" />

<BR>

**iii. Schema** <br>

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
  
✔ Built using Surrogate keys, Clean joins and Standardized attributes

<br>
<br>

#### 3. Data Quality Testing
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

#### 4. Source Freshness Monitoring
Source freshness checks were configured to simulate production monitoring:

```yaml
freshness:
  warn_after: {count: 3650, period: day}
  error_after: {count: 7300, period: day}
```

<br>
<br>

### Analytics Marts
#### 1. Sales Analytics

- sales_daily
  - Daily revenue, orders, items sold, discounts

- sales_by_store
  - Store-level performance and ranking

- sales_by_product
  - Product/category performance insights

<br>

#### 2. Customer Analytics

- customer_lifetime_value
  - Revenue contribution
  - Order behavior
  - Customer tenure

- customer_rfm
  - Recency
  - Frequency
  - Monetary value

✔ Enables customer segmentation and behavioral analysis

<br>
<br>

### Business Intelligence (Power BI)
A multi-page dashboard was developed to support business decision-making:

- Page 1: Executive Overview
  - Total Revenue, Orders, Discount and Gross Sales KPI
  - Revenue & order trends over time

<Br>

- Page 2: Store Performance
  - Revenue by store
  - Order distribution
  - Store records

<br>

- Page 3: Product Performance
  - Category Performance
  - Top-performing products
  - Revenue vs Unit sold

<br>

- Page 4: Customer Insights
  - Customer records
  - Customer segmentation (RFM)
  - Purchase distribution


<br>
<br>

## Key Insights

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
