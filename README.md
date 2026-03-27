# 🛒 Sales EDA – SQL Server Project

## Overview

This project is an **Exploratory Data Analysis (EDA)** built entirely in **SQL Server**.  
It explores sales, customer, and product data using structured queries and reusable views.

The goal is to extract meaningful business insights — such as customer segmentation, purchase behavior, and sales performance — directly from raw relational data.

---

## Database Structure

The project uses a **star schema** with three core tables:

| Table | Description |
|---|---|
| `fact_sales` | Transactional data — orders, quantities, and revenue |
| `dim_customers` | Customer master data — demographics and identifiers |
| `dim_products` | Product master data — product details and keys |

---

## Views

Two reusable SQL views were created as the analytical layer of this project:

- **`report_customers`** — Customer-level aggregation with segmentation logic
- **`report_products`** — Product-level aggregation with performance metrics

> These views serve as the **data source for Power BI dashboards** (in progress).

---

## View 1: `report_customers`

### Purpose
Aggregates all sales activity per customer and enriches it with behavioral metrics, age categorization, and business segmentation labels.

---

### Step 1 — Base Query (CTE: `base_query`)

```sql
WITH base_query AS (
    SELECT 
        f.order_number,
        f.product_key,
        f.order_date,
        f.sales_amount,
        f.quantity,
        c.customer_key,
        c.customer_number,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        DATEDIFF(YEAR, c.birthdate, GETDATE()) AS age
    FROM fact_sales f
    LEFT JOIN dim_customers c ON f.customer_key = c.customer_key
    WHERE order_date IS NOT NULL
)
```

**What this does:**

- Joins `fact_sales` with `dim_customers` using a `LEFT JOIN` to keep all sales records, even if customer data is incomplete.
- Combines `first_name` and `last_name` into a single readable `customer_name` field.
- Calculates each customer's current **age** using `DATEDIFF` between their birthdate and today.
- Filters out rows where `order_date` is `NULL` to ensure only valid transactions are analyzed.

---

### Step 2 — Customer Aggregation (CTE: `customer_aggregation`)

```sql
customer_aggregation AS (
    SELECT
        customer_key,
        customer_number,
        customer_name,
        age,
        COUNT(order_number)          AS total_orders,
        SUM(sales_amount)            AS total_sales,
        SUM(quantity)                AS total_quantity,
        COUNT(DISTINCT product_key)  AS total_products,
        MAX(order_date)              AS last_order_date,
        DATEDIFF(MONTH, MIN(order_date), MAX(order_date)) AS lifespan
    FROM base_query
    GROUP BY customer_key, customer_number, customer_name, age
)
```

**What this does:**

- Groups all transactions by customer to produce one summary row per customer.
- Calculates the following aggregated metrics:

| Metric | Description |
|---|---|
| `total_orders` | Number of orders placed |
| `total_sales` | Total revenue generated |
| `total_quantity` | Total units purchased |
| `total_products` | Number of unique products bought |
| `last_order_date` | Date of most recent purchase |
| `lifespan` | Months between first and last purchase — measures customer loyalty window |

---

### Step 3 — Final SELECT with Business Logic

```sql
SELECT 
    customer_key,
    customer_number,
    customer_name,
    age,
    total_orders,
    total_sales,
    total_quantity,
    total_products,
    last_order_date,
    lifespan,

    DATEDIFF(MONTH, last_order_date, GETDATE()) AS recency,

    CASE 
        WHEN age < 30                  THEN 'Below 30'
        WHEN age BETWEEN 30 AND 39     THEN '30–39'
        WHEN age BETWEEN 40 AND 49     THEN '40–49'
        WHEN age BETWEEN 50 AND 59     THEN '50–59'
        ELSE '60 and above'
    END AS age_category,

    CASE 
        WHEN lifespan >= 12 AND total_sales > 5000  THEN 'VIP'
        WHEN lifespan >= 12 AND total_sales <= 5000 THEN 'Regular'
        ELSE 'New'
    END AS customer_segment,

    CASE 
        WHEN lifespan = 0 THEN total_sales
        ELSE total_sales / lifespan
    END AS monthly_avg_sales,

    CASE 
        WHEN total_orders = 0 THEN 0
        ELSE total_sales / total_orders
    END AS avg_order_value

FROM customer_aggregation;
```

**What this does — field by field:**

**`recency`**  
Calculates how many months have passed since the customer's last purchase. A low recency value means the customer is still active and engaged.

**`age_category`**  
Groups customers into demographic age bands using a `CASE` statement. Useful for age-based segmentation in marketing and reporting.

**`customer_segment`**  
Classifies each customer into one of three business segments based on loyalty and revenue:
- **VIP** — Active for 12+ months AND generated over $5,000 in sales
- **Regular** — Active for 12+ months but below $5,000
- **New** — Customer lifespan under 12 months

**`monthly_avg_sales`**  
Average revenue generated per month. A `CASE` guard prevents division by zero for customers whose first and last order were in the same month (`lifespan = 0`).

**`avg_order_value`**  
Average revenue per order. Another division-by-zero guard is applied in case `total_orders` is 0.

---

## View 2: `report_products`

### Purpose
Aggregates all sales activity per product and enriches it with profitability metrics, performance classification, and customer reach indicators.

---

### Step 1 — Base Query (CTE: `base_query`)

```sql
WITH base_query AS (
    SELECT 
        f.order_number,
        f.order_date,
        f.sales_amount,
        f.quantity,
        f.customer_key,
        p.product_key,
        p.product_name,
        p.category,
        p.subcategory,
        p.cost
    FROM fact_sales f
    LEFT JOIN dim_products p ON f.product_key = p.product_key
    WHERE order_date IS NOT NULL
)
```

**What this does:**

- Joins `fact_sales` with `dim_products` using a `LEFT JOIN` to retain all sales records regardless of whether product data is fully populated.
- Pulls both transactional fields (`sales_amount`, `quantity`) and product descriptors (`category`, `subcategory`, `cost`) into a single flat dataset.
- Filters out rows where `order_date` is `NULL` to ensure only valid transactions are included.

---

### Step 2 — Product Aggregation (CTE: `product_aggregations`)

```sql
product_aggregations AS (
    SELECT 
        product_key,
        product_name,
        category,
        subcategory,
        cost,
        COUNT(DISTINCT order_number)                                    AS total_orders,
        MAX(order_date)                                                 AS last_order_date,
        DATEDIFF(MONTH, MIN(order_date), MAX(order_date))               AS lifespan,
        SUM(cost)                                                       AS total_cost,
        SUM(sales_amount)                                               AS total_sales,
        SUM(quantity)                                                   AS total_quantity,
        COUNT(DISTINCT customer_key)                                    AS total_customers,
        ROUND(AVG(CAST(sales_amount AS FLOAT) / NULLIF(quantity, 0)), 2) AS avg_selling_price
    FROM base_query
    GROUP BY product_key, product_name, category, subcategory, cost
)
```

**What this does — field by field:**

| Metric | Description |
|---|---|
| `total_orders` | Number of distinct orders containing this product |
| `last_order_date` | Date the product was most recently purchased |
| `lifespan` | Months between first and last sale — measures how long the product has been actively selling |
| `total_cost` | Sum of unit costs across all sold units — used to estimate total expenditure |
| `total_sales` | Total revenue generated by the product |
| `total_quantity` | Total units sold |
| `total_customers` | Number of unique customers who purchased this product |
| `avg_selling_price` | Average price per unit sold. `CAST` to `FLOAT` ensures decimal precision; `NULLIF(quantity, 0)` prevents division by zero |

---

### Step 3 — Final SELECT with Business Logic

```sql
SELECT 
    product_key,
    avg_selling_price,     
    product_name,
    category,
    subcategory,
    total_orders,
    last_order_date,
    lifespan,
    cost,
    total_cost,
    total_sales,

    CASE
        WHEN total_sales < 100000                        THEN 'Low Performance'
        WHEN total_sales BETWEEN 100000 AND 500000       THEN 'Mid Range'
        ELSE                                                  'High Performance'
    END AS product_performance,

    total_quantity,
    total_customers,
    avg_selling_price,

    DATEDIFF(MONTH, last_order_date, GETDATE()) AS recency,

    CASE 
        WHEN total_orders = 0 THEN 0
        ELSE total_sales / total_orders
    END AS avg_order_revenue,

    CASE 
        WHEN lifespan = 0 THEN total_sales
        ELSE total_sales / lifespan
    END AS avg_monthly_revenue

FROM product_aggregations;
```

**What this does — field by field:**

**`product_performance`**  
Classifies each product into a performance tier based on total revenue generated:
- **Low Performance** — under $100,000 in total sales
- **Mid Range** — between $100,000 and $500,000
- **High Performance** — above $500,000

**`recency`**  
Months since the product's last sale. Helps identify products that may be declining or no longer in demand.

**`avg_order_revenue`**  
Average revenue per order for this product. Division-by-zero guard applied for products with no recorded orders.

**`avg_monthly_revenue`**  
Average revenue generated per month over the product's selling lifespan. For products sold within a single month (`lifespan = 0`), the total sales value is used directly to avoid division by zero.

---


## Tools Used

- **SQL Server** — Query engine and view creation
- **Power BI** *(in progress)* — Dashboard and visualization layer connected to the views

---

## Author

**Andranik** — Data Analyst  
Marketing background with hands-on experience in SQL, Python, Power BI, and Excel.  
Open to data analyst opportunities.

📎 [LinkedIn](#) <!-- Replace with your LinkedIn URL -->
