# Step-by-Step Guide: How I Built the Power BI Data Model

This guide documents every step I followed, from project rules to Row-Level Security.

---

## Core Rules (Set Before Touching Any Data)

1. **Target schema:** Star / Galaxy schema, with facts in the middle and dimensions around them
2. **Never connect fact tables directly to each other.** Connect them only through shared dimensions
3. **Filter direction:** dimension to fact, one-to-many, single direction
4. **State the grain** of every table before transforming it, to prevent grain mismatches and row fanning
5. **Every column must earn its place.** Remove technical columns, unused source IDs, descriptions and hash keys
6. **Protect the numbers.** Pin a baseline total (e.g. `line_total` in a card visual) and re-check it after every merge
7. **Naming standards:** `snake_case`, `fact_` / `dim_` prefixes, and `_key` only for newly created surrogate keys (e.g. `product_key` vs `product_code`)

---

## Phase 1: Preparation, Exploration and Workspace Setup

### Step 1: Understand the business entities
- Open **Table view** and explore every table
- Classify them into **candidate dimensions** (e.g. `customer_master`, `products`, `cities`) and **candidate facts/events** (e.g. `orders`, `order_line_items`, `invoices`, `campaign_log`)

### Step 2: Spot data anomalies early
- Duplicate tables (e.g. `sheet1` duplicating shipments)
- Raw, unpromoted headers
- Disconnected single-column junk tables (e.g. `dimension order`)
- Inconsistent terminology (e.g. "users" vs "customers")

### Step 3: Organise the Power Query workspace
Create numbered query groups:

| Group | Purpose |
|---|---|
| `01_stage` | All raw source queries as ingested |
| `02_dimensions` | Transformed dimension queries |
| `03_facts` | Transformed fact queries |
| `04_support` | Auxiliary tables (security mapping, manual translation tables) |
| `other_queries` | Inbox for newly connected tables to be evaluated |

---

## Phase 2: Building Clean Dimensions

Always create dimensions by **referencing** stage tables, never by editing the raw staging queries.

### Dimension 1: `dim_customer`

1. **Base:** Reference `customer_master` into `02_dimensions` and rename it `dim_customer`
2. **Clean:** Filter out dummy/test rows (e.g. customer ID 999). Baseline: **60 unique customers**
3. **Merge contacts without fanning out:**
   - Customers have multiple contact persons, so a direct merge duplicates customer rows
   - Filter `customer_contacts` where `is_primary = TRUE` first, then merge
   - Verify with **Group By, Count Rows** that the 60-row grain is preserved
4. **Merge user details** on user ID to bring in credit limit and phone number. Use "customer" as the standard business term instead of mixing "user" and "customer"
5. **Denormalise the address** (avoid a snowflake schema):
   - Verify cardinality first (1:1 or 1:many from the dimension)
   - Left-join addresses on `address_id` and extract street and city
   - Promote headers on cities, left-join, and extract region
   - Drop the standalone `regions` table, since region is already included
6. **Housekeeping:** Remove hash keys, source IDs, address IDs and redundant contact indicators. Rename columns to `snake_case`
7. **Disable load** on the consolidated raw queries (`addresses`, `customer_master`, `cities`, `customer_contacts`, `regions`, `user_details`)

### Dimension 2: `dim_product`

1. **Base:** Reference `products` into `02_dimensions`, rename to `dim_product`, and remove test product codes
2. **Enrich with category:**
   - In `subcategories`, promote headers
   - Split the combined category/subcategory column by the pipe (`|`) delimiter into two columns
   - Apply **Capitalize Each Word** to subcategory text so the casing matches `products` (otherwise the merge returns nulls)
   - Left-join subcategories into `dim_product` to extract the category
3. **Surrogate key:** Source records use business product codes rather than numeric IDs, so add an **Index Column** starting from 1 named `product_key`
4. **Refine:** Drop large description text and source hash keys, apply proper case, order columns logically, and disable load on raw `products` and `subcategories`

### Dimension 3: `dim_order_flags` (Junk Dimension)

1. Reference `orders` and keep only `order_channel`, `status` and `priority`
2. Remove duplicates to get a compact **14-row** junk dimension
3. Translate the cryptic channel codes using **Enter Data** to create a `channels` table:
   - 10 = Online Store
   - 20 = Retail Partner
   - 30 = Wholesale
   - 40 = Field Sale
4. Left-join `channels` into `dim_order_flags` to replace codes with business-friendly text
5. Add an index column named `flag_key` and rename columns to `snake_case`

### Dimension 4: `dim_geo` (Role-Playing Dimension)

1. Reference the raw `cities` table
2. Remove duplicates
3. Add an index column named `geo_key`
4. Rename columns to `city` and `region`
5. This dimension plays two roles: **ship-to** and **bill-to** location

### Dimension 5: `dim_campaign`

1. Reference `campaign_log`
2. Keep static attributes only: `campaign_name`, `channel`, `start_date`, `end_date`, `budget`
3. Remove duplicates
4. Add an index column named `campaign_key`

---

## Phase 3: Building Facts and Connecting Relationships

### Fact 1: `fact_sales` (Header vs Detail Transactions)

**Principle:** Headers store context once, while detail lines store the measures at the lowest grain. Never link header facts to detail facts through an order key in the model. Build the fact at the finest level (the lines), because numbers can always roll up.

1. **Append historical tables:** Append `orders_2025` and `orders_2026` into a single intermediate `orders` staging query, removing legacy null columns and notes
2. **Baseline:** Reference `order_line_items` as the base of `fact_sales`
3. **Set the baseline number:** In Report view, add `line_total` to a card visual and note the total
4. **Bring in header context:** Left-join the appended `orders` on `order_id`. Confirm the total sales did not change
5. **Dimension lookups:**
   - Left-join `dim_customer` (by customer name) to get `customer_id`
   - Left-join `dim_product` (by product name) to get `product_key`
   - Left-join `dim_order_flags` on all three matching attributes to get `flag_key`
   - Left-join `dim_geo` **twice**: on `ship_to_city` (giving `ship_to_city_key`) and on `bill_to_city` (giving `bill_to_city_key`)
6. **Troubleshoot fan-out:** If sales total spikes after merging `dim_product`, look for duplicate product codes. In `dim_product`, filter out incomplete duplicate records (missing brand/price), rebuild the index, and confirm the original total is restored
7. **Drop redundant columns:**
   - Remove customer city/region from the fact so `dim_customer` is the single source of truth
   - Keep `ship_to_city` and `bill_to_city` as keys, since they vary per order
   - Drop pre-aggregated header `order_total` columns to avoid discrepancies. Rely on the sum of line totals
8. **Column order:** surrogate foreign keys first, transaction IDs second, dates third, numeric measures last

### Fact 2: `fact_inventory` (Periodic Snapshot)

1. Raw data stores months as columns, which is human-friendly but bad for modeling
2. Select the product columns and run **Unpivot Other Columns** to create `month` and `units`
3. Convert month strings to a proper date type
4. Merge `dim_product` to look up `product_key`

### Fact 3: `fact_sales_targets`

1. Reference the raw budget data
2. Convert headers into `date` and `target_revenue`
3. Keep it as a lightweight target fact

### Fact 4: `fact_order_process` (Accumulating Snapshot)

**Why:** Order, shipment, delivery, invoice and payment share one lifecycle and the same amounts. Five separate fact tables would cause confusion and duplicated measures.

1. Reference the consolidated `orders` spine
2. Look up `customer_id` through `dim_customer`
3. Left-join `shipments`, `invoices` and `payments` in sequence by business keys
4. Result: one row per order with `order_date`, `shipment_date`, `delivery_date`, `invoice_date` and `payment_date`
5. Disable load on the redundant tables (`invoices`, `invoice_lines`, `payments`, `shipments`)

### Fact 5: `fact_promotion_coverage` (Factless Fact for Many-to-Many)

1. In `campaign_skus`, promote headers
2. Split the comma-delimited product code column: **Split Column by Delimiter, Advanced, Rows**
3. Trim whitespace
4. Look up `campaign_key` and `product_key`, then drop the text names
5. Result: a factless fact with no numeric measures, acting as a coverage bridge between campaigns and products

### Configure Relationships in Model View

1. Arrange **dimensions on the perimeter** and **facts in the centre**
2. Connect every dimension to its facts: **1:\*** cardinality, **single** filter direction (dimension to fact)
3. **Role-playing `dim_geo`:**
   - `geo_key` to `ship_to_city_key`: **active**
   - `geo_key` to `bill_to_city_key`: **inactive**, activated in DAX with `USERELATIONSHIP`
4. Delete junk queries (`sheet1`, `dimension order`) and disable load on all intermediate and staging queries

---

## Phase 4: Polish, DAX, Shared Dimensions and Security

### Step 1: Formatting, data types and summarisation
- Confirm all table and column names follow `snake_case`
- Standardise date formats across all tables (e.g. `YYYY-MM-DD`)
- Set identifiers, codes and budgets (where sums make no sense) to **Don't Summarize**

### Step 2: Build the central `dim_date`

```DAX
dim_date = CALENDARAUTO()
```

- `CALENDARAUTO()` scans the model for the earliest and latest dates and builds a contiguous range that grows with the data
- Add calculated columns such as `year = YEAR(dim_date[Date])` and `month = MONTH(dim_date[Date])`
- Connect `dim_date[Date]` to the date key in `fact_sales`, `fact_sales_targets`, `fact_inventory`, `fact_campaign_spend` and `fact_order_process`
- This shared date axis allows actual sales vs targets vs inventory comparisons

### Step 3: Centralised DAX measures

Create an empty table named `_measures` and store all explicit measures there.

```DAX
total_sales = SUM(fact_sales[line_total])
```
Format as a whole number with thousands separators.

```DAX
total_orders = DISTINCTCOUNT(fact_sales[order_id])
```
Use `DISTINCTCOUNT`, not `COUNT`, because there are multiple rows per order line.

```DAX
total_active_customers = DISTINCTCOUNT(fact_sales[customer_id])
total_customers = COUNT(dim_customer[customer_id])
```
These separate customers who actually ordered from all registered customers.

Process duration, as a calculated column in `fact_order_process`:

```DAX
order_to_pay = DATEDIFF(fact_order_process[order_date], fact_order_process[payment_date], DAY)
```

Then the measure:

```DAX
avg_order_to_pay = AVERAGE(fact_order_process[order_to_pay])
```

### Step 4: Dynamic Row-Level Security

1. **Clean the security table:** Move it to `04_support`, format headers as `user_email` and `region`
2. **Relationship:** Connect `security[region]` to `dim_customer[region]` as 1:\*, single direction (security to customer). Because it connects to `dim_customer`, security cascades to `fact_sales` and `fact_order_process` without altering their grain
3. **Define the role:** Under **Manage Roles**, create `regional_access` on `dim_customer` with:

```DAX
[region] = LOOKUPVALUE(security[region], security[user_email], USERPRINCIPALNAME())
```

4. **Validate:** Use **View As** in Power BI Desktop and enter an employee email. Confirm cards and customer regions show only the authorised geography, and that all data returns when the simulation is stopped

---

## Final Checklist

- [ ] Baseline `total_sales` unchanged after every merge
- [ ] 60 rows in `dim_customer`, with no duplicates in any dimension key
- [ ] All relationships are 1:\* single direction, dimension to fact
- [ ] Only one active relationship path between any two tables
- [ ] Load disabled on all staging and intermediate queries
- [ ] All names follow `snake_case` with `fact_` / `dim_` prefixes
- [ ] All measures stored in `_measures`
- [ ] RLS tested with View As
