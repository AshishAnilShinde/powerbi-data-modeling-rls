# Power BI Data Modeling Project: Star/Galaxy Schema with Row-Level Security

An end-to-end Power BI data modeling project that takes raw, messy source tables and turns them into a clean, analysis-ready **galaxy schema** (multiple fact tables sharing conformed dimensions), complete with centralised DAX measures, a shared date dimension, and **dynamic Row-Level Security (RLS)**.

> The focus of this project is the **data model**, not the dashboard. Every decision follows industry best practices for modeling in Power Query and Power BI.

---

## Project Highlights

- **Galaxy schema** with 4 core dimensions plus a junk dimension, a role-playing dimension and a shared date dimension
- **Multiple fact types:** transactional, periodic snapshot, accumulating snapshot, target and factless fact tables
- **Power Query best practices:** numbered query groups, staging layer, load disabled on intermediate queries
- **Grain-aware modeling:** fan-out issues identified, diagnosed and fixed
- **Dynamic RLS** using `USERPRINCIPALNAME()` and a security mapping table
- **Centralised measures** in a dedicated `_measures` table

---

## Core Modeling Principles

| Principle | What it means in this project |
|---|---|
| Star / Galaxy schema | Facts in the centre, dimensions around them. Fact tables are never connected directly to each other |
| Filter direction | Always dimension to fact, one-to-many, single direction |
| Explicit grain | The grain (what one row represents) is stated for every table before transforming it |
| Every column earns its place | Technical columns, unused IDs, descriptions and hash keys are removed |
| Protect the numbers | A baseline total (e.g. total line sales) is verified after every merge |
| Naming standards | `snake_case`, `fact_` / `dim_` prefixes, `_key` suffix only for surrogate keys (e.g. `product_key` vs `product_code`) |

---

## Data Model Overview

### Dimensions

| Table | Purpose |
|---|---|
| `dim_customer` | One row per customer (60 unique), with primary contact, credit details and flattened address/region |
| `dim_product` | Products enriched with category and subcategory, with surrogate `product_key` |
| `dim_order_flags` | **Junk dimension** combining order channel, status and priority (14 rows) with surrogate `flag_key` |
| `dim_geo` | **Role-playing dimension** used for both ship-to and bill-to cities, with `geo_key` |
| `dim_campaign` | Static campaign attributes with `campaign_key` |
| `dim_date` | Central date table shared by all facts |

### Facts

| Table | Type | Grain |
|---|---|---|
| `fact_sales` | Transactional | One row per order line item |
| `fact_inventory` | Periodic snapshot | Product per month |
| `fact_sales_targets` | Target fact | Target revenue per date |
| `fact_order_process` | Accumulating snapshot | One row per order, with all milestone dates |
| `fact_promotion_coverage` | Factless fact | Campaign and product coverage (bridge) |

### Relationships

- All dimensions connect to facts as **1:\*** with **single** filter direction (dimension to fact)
- `dim_geo` plays two roles:
  - `geo_key` to `ship_to_city_key`: **active**
  - `geo_key` to `bill_to_city_key`: **inactive**, activated inside DAX measures using `USERELATIONSHIP`
- `dim_date[Date]` connects to the date key of every fact, enabling cross-fact comparison (actual vs target vs inventory)
- The `security` table connects to `dim_customer[region]` (1:\*, single direction) and cascades down to the facts

---

## Power Query Organisation

Queries are grouped into numbered folders:

| Folder | Contents |
|---|---|
| `01_stage` | Raw source queries as ingested (load disabled) |
| `02_dimensions` | Transformed dimension queries |
| `03_facts` | Transformed fact queries |
| `04_support` | Auxiliary tables (security mapping, manual translation tables) |
| `other_queries` | Inbox for newly connected tables awaiting review |

Dimensions and facts are built by **referencing** stage queries, so raw staging queries are never modified.

---

## DAX Measures

All measures live in an empty `_measures` table.

```DAX
total_sales = SUM(fact_sales[line_total])

total_orders = DISTINCTCOUNT(fact_sales[order_id])

total_active_customers = DISTINCTCOUNT(fact_sales[customer_id])

total_customers = COUNT(dim_customer[customer_id])

avg_order_to_pay = AVERAGE(fact_order_process[order_to_pay])
```

Calculated column in `fact_order_process`:

```DAX
order_to_pay = DATEDIFF(fact_order_process[order_date], fact_order_process[payment_date], DAY)
```

Date table:

```DAX
dim_date = CALENDARAUTO()
```

---

## Row-Level Security

Dynamic RLS restricts each user to their assigned region.

- Role: `regional_access` on `dim_customer`
- Filter expression:

```DAX
[region] = LOOKUPVALUE(security[region], security[user_email], USERPRINCIPALNAME())
```

- The security table filters `dim_customer`, which cascades to `fact_sales` and `fact_order_process` without changing their grain
- Validated using **View As** in Power BI Desktop

---

## Key Challenges Solved

| Challenge | Fix |
|---|---|
| Customer rows fanning out after merging contacts | Filtered `customer_contacts` to `is_primary = TRUE` before merging to keep the 60-row grain |
| Merge returning nulls on subcategory | Standardised text casing (Capitalize Each Word) before joining |
| Sales total spiking after merging `dim_product` | Found duplicate product codes, removed incomplete duplicate records, rebuilt the index |
| Raw monthly inventory columns | Unpivoted other columns into `month` and `units` rows |
| Many-to-many between campaigns and products | Exploded comma-delimited SKUs into rows to create a factless fact table |
| Same city used as ship-to and bill-to | Role-playing dimension with one active and one inactive relationship |

---

## Tools Used

- Power BI Desktop (Power Query, Model view, DAX)
- Power BI Row-Level Security (View As validation)

---

## Documentation

See [`STEPS.md`](./STEPS.md) for the full step-by-step guide to how this model was built.

---

## Author

**Ashish Shinde**
Add your GitHub / LinkedIn links here.
