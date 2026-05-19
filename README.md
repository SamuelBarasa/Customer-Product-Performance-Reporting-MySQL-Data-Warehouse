# Customer & Product Report Views — MySQL

## Overview

This project creates two reusable **MySQL Views** that consolidate and summarize key business metrics from a sales data warehouse. The views are built using **Common Table Expressions (CTEs)** and **window-style aggregations**, making them easy to query for dashboards, BI tools, or further analysis.

| View | Purpose |
|---|---|
| `report_customers` | Customer-level metrics, segmentation, and KPIs |
| `report_products` | Product-level metrics, segmentation, and KPIs |

---

## Database Schema

The views draw from the following tables:

### `fact_sales`
| Column | Description |
|---|---|
| `order_number` | Unique order identifier |
| `product_key` | Foreign key to `dim_products` |
| `customer_key` | Foreign key to `dim_customers` |
| `order_date` | Date the order was placed |
| `sales_amount` | Revenue generated from the order |
| `quantity` | Number of units sold |

### `dim_customers`
| Column | Description |
|---|---|
| `customer_key` | Primary key |
| `customer_number` | Business-facing customer ID |
| `first_name` | Customer first name |
| `last_name` | Customer last name |
| `birthdate` | Customer date of birth |

### `dim_products`
| Column | Description |
|---|---|
| `product_key` | Primary key |
| `product_name` | Name of the product |
| `category` | Top-level product category |
| `subcategory` | Product subcategory |
| `cost` | Product cost price |

---

## View 1: `report_customers`

### Purpose
Consolidates customer transaction history and computes behavioural metrics and segments to support customer analytics and CRM use cases.

### How It Works — Query Logic

```
fact_sales  ──INNER JOIN──  dim_customers
        │
        ▼
   base_query          (row-level detail)
        │
        ▼
customer_aggregation   (grouped by customer)
        │
        ▼
Final SELECT           (segments + KPIs)
```

**CTE 1 — `base_query`**
Joins `fact_sales` with `dim_customers` and pulls row-level transaction details. Also computes the customer's current **age** using `TIMESTAMPDIFF`.

**CTE 2 — `customer_aggregation`**
Groups by customer and computes:
- Total orders, sales, quantity, and distinct products
- First and last order dates
- Customer lifespan in months

**Final SELECT**
Adds derived segments and KPIs on top of the aggregated data.

### Output Columns

| Column | Type | Description |
|---|---|---|
| `customer_key` | INT | Primary customer identifier |
| `customer_number` | VARCHAR | Business-facing customer ID |
| `customer_name` | VARCHAR | Full name (first + last) |
| `age` | INT | Current age in years |
| `age_group` | VARCHAR | Age band segment (see below) |
| `customer_segment` | VARCHAR | VIP / Regular / New (see below) |
| `last_order_date` | DATE | Date of most recent purchase |
| `recency` | INT | Months since last order |
| `total_orders` | INT | Count of distinct orders |
| `total_sales` | DECIMAL | Total revenue generated |
| `total_quantity` | INT | Total units purchased |
| `total_products` | INT | Count of distinct products bought |
| `lifespan` | INT | Months between first and last order |
| `avg_order_value` | DECIMAL | Average revenue per order |
| `avg_monthly_spend` | DECIMAL | Average revenue per active month |

### Segmentation Logic

**Age Groups**
| Segment | Condition |
|---|---|
| `Under 20` | age < 20 |
| `20-29` | age BETWEEN 20 AND 29 |
| `30-39` | age BETWEEN 30 AND 39 |
| `40-49` | age BETWEEN 40 AND 49 |
| `50 and above` | age >= 50 |

**Customer Segments**
| Segment | Condition |
|---|---|
| `VIP` | lifespan >= 12 months AND total_sales > 5,000 |
| `Regular` | lifespan >= 12 months AND total_sales <= 5,000 |
| `New` | lifespan < 12 months |

### KPI Formulas

```sql
-- Average Order Value
CASE WHEN total_orders = 0 THEN 0
     ELSE total_sales / total_orders
END AS avg_order_value

-- Average Monthly Spend
CASE WHEN lifespan = 0 THEN total_sales
     ELSE total_sales / lifespan
END AS avg_monthly_spend
```

---

## View 2: `report_products`

### Purpose
Summarizes product sales performance and assigns performance segments to support product analytics, inventory planning, and marketing prioritization.

### How It Works — Query Logic

```
fact_sales  ──INNER JOIN──  dim_products
        │
        ▼
   base_query              (row-level detail)
        │
        ▼
product_aggregations       (grouped by product)
        │
        ▼
Final SELECT               (segments + KPIs)
```

**CTE 1 — `base_query`**
Joins `fact_sales` with `dim_products` and retrieves row-level transaction data alongside product attributes.

**CTE 2 — `product_aggregations`**
Groups by product and computes:
- Total orders, customers, sales, and quantity
- First and last sale dates
- Product lifespan in months
- Average selling price (guarded against division by zero with `NULLIF`)

**Final SELECT**
Adds performance segments and revenue KPIs.

### Output Columns

| Column | Type | Description |
|---|---|---|
| `product_key` | INT | Primary product identifier |
| `product_name` | VARCHAR | Product name |
| `category` | VARCHAR | Top-level category |
| `subcategory` | VARCHAR | Product subcategory |
| `cost` | DECIMAL | Product cost price |
| `last_sale_date` | DATE | Date of most recent sale |
| `recency_in_months` | INT | Months since last sale |
| `product_segment` | VARCHAR | High-Performer / Midrange / Low-Performer |
| `lifespan` | INT | Months between first and last sale |
| `total_orders` | INT | Count of distinct orders |
| `total_sales` | DECIMAL | Total revenue generated |
| `total_quantity` | INT | Total units sold |
| `total_customers` | INT | Count of distinct customers |
| `avg_selling_price` | DECIMAL | Average price per unit sold |
| `avg_order_revenue` | DECIMAL | Average revenue per order |
| `avg_monthly_revenue` | DECIMAL | Average revenue per active month |

### Segmentation Logic

**Product Segments**
| Segment | Condition |
|---|---|
| `High-Performer` | total_sales > 50,000 |
| `Midrange` | total_sales BETWEEN 10,000 AND 50,000 |
| `Low-Performer` | total_sales < 10,000 |

### KPI Formulas

```sql
-- Average Selling Price (safe division)
ROUND(AVG(CAST(sales_amount AS FLOAT) / NULLIF(quantity, 0)), 1) AS avg_selling_price

-- Average Order Revenue
CASE WHEN total_orders = 0 THEN 0
     ELSE total_sales / total_orders
END AS avg_order_revenue

-- Average Monthly Revenue
CASE WHEN lifespan = 0 THEN total_sales
     ELSE total_sales / lifespan
END AS avg_monthly_revenue
```

---

## Usage

### Create the Views
Run the full SQL script in MySQL Workbench or any MySQL client using **Ctrl + A** then execute to ensure both views are created together.

```sql
-- Query the customer report
SELECT * FROM report_customers;

-- Query the product report
SELECT * FROM report_products;
```

### Example Queries

```sql
-- Find all VIP customers ordered by total sales
SELECT customer_name, total_sales, recency
FROM report_customers
WHERE customer_segment = 'VIP'
ORDER BY total_sales DESC;

-- Find top performing products by category
SELECT category, product_name, total_sales, product_segment
FROM report_products
WHERE product_segment = 'High-Performer'
ORDER BY total_sales DESC;

-- Average spend by age group
SELECT age_group, ROUND(AVG(total_sales), 2) AS avg_sales
FROM report_customers
GROUP BY age_group
ORDER BY avg_sales DESC;

-- Products with no recent sales (inactive for 6+ months)
SELECT product_name, last_sale_date, recency_in_months
FROM report_products
WHERE recency_in_months >= 6
ORDER BY recency_in_months DESC;
```

---

## Requirements

| Requirement | Detail |
|---|---|
| MySQL Version | **8.0+** (CTEs require MySQL 8.0) |
| Tables needed | `fact_sales`, `dim_customers`, `dim_products` |
| Permissions | `CREATE VIEW` privilege on the target database |

---

## Notes

- Both views use `INNER JOIN` to ensure only matched records between fact and dimension tables are included, preventing `NULL` values in key columns.
- Division-by-zero is handled explicitly using `CASE WHEN ... = 0` guards and `NULLIF()` where applicable.
- `TIMESTAMPDIFF(MONTH, ...)` and `CURDATE()` are MySQL-native functions — this script is **not compatible** with SQL Server or Oracle without modification.
- Views are **not materialised** — they re-execute the full query on every call. For large datasets, consider creating indexed summary tables instead.
