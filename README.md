# Amazon Marketplace SQL Project

A fully synthetic, relationally consistent Amazon-style marketplace dataset built for SQL practice and portfolio demonstration.
10 interconnected tables, ~500K rows, zero NULLs, zero orphan records — verified against a live PostgreSQL 16 instance.


## Contents
schema.sql             PostgreSQL DDL: tables, PKs, FKs, CHECK constraints, indexes
er_diagram.mmd          Mermaid ER diagram
csv/                    10 CSV files, ready for COPY import
validation_report.md    Full validation results (zero errors)
README.md               This file


## Import order
Run in this exact order to satisfy foreign key dependencies:
| # | Table | Depends on |
|---|-------|------------|
| 1 | `category` | — |
| 2 | `sellers` | — |
| 3 | `customers` | — |
| 4 | `products` | category, sellers |
| 5 | `inventory` | products |
| 6 | `orders` | customers |
| 7 | `order_items` | orders, products |
| 8 | `payments` | orders |
| 9 | `shipping` | orders |
| 10 | `reviews` | products, customers |


---

## Dataset sizes

| Table | Rows |
|---|---|
| category | 25 |
| sellers | 500 |
| customers | 15,000 |
| products | 8,000 |
| inventory | 8,000 |
| orders | 60,000 |
| order_items | 180,000 |
| payments | 60,000 |
| shipping | 60,000 |
| reviews | 120,000 |
| **Total** | **511,525** |

---

## Bussiness Problems Solved

1. Business Scenario: Growth wants to measure month-over-month customer retention — the percentage of customers who ordered in one month and also ordered in the following month.

Question: Calculate the month-over-month retention rate: for each month, what percentage of that month's customers also placed an order the following month?

Return: order_month, retained_customers, total_customers, retention_rate

```sql
WITH monthly_customers AS (
    SELECT DISTINCT
        DATE_TRUNC('month', order_date)::date AS order_month,
        customer_id
    FROM orders
),
retention AS (
    SELECT
        curr.order_month,
        curr.customer_id,
        CASE WHEN next.customer_id IS NOT NULL THEN 1 ELSE 0 END AS retained
    FROM monthly_customers curr
    LEFT JOIN monthly_customers next
        ON curr.customer_id = next.customer_id
        AND next.order_month = curr.order_month + INTERVAL '1 month'
)
SELECT
    order_month,
    SUM(retained) AS retained_customers,
    COUNT(*) AS total_customers,
    ROUND(100.0 * SUM(retained) / COUNT(*), 2) AS retention_rate
FROM retention
GROUP BY order_month
ORDER BY order_month;
```

2. Business Scenario: Supply chain wants to proactively flag products at high risk of stockout based on current stock levels versus recent sales velocity.

Question: Identify products where current stock_quantity would be depleted within 7 days based on their average daily units sold over the last 30 days.

Return: product_name, warehouse, stock_quantity, avg_daily_units_sold, days_of_stock_remaining

```sql
WITH sales_velocity AS (
    SELECT
        oi.product_id,
        SUM(oi.quantity) / 30.0 AS avg_daily_units_sold
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.order_date >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY oi.product_id
)
SELECT
    p.product_name,
    i.warehouse,
    i.stock_quantity,
    ROUND(sv.avg_daily_units_sold, 2) AS avg_daily_units_sold,
    ROUND(i.stock_quantity / NULLIF(sv.avg_daily_units_sold, 0), 1) AS days_of_stock_remaining
FROM inventory i
JOIN products p ON i.product_id = p.product_id
JOIN sales_velocity sv ON sv.product_id = p.product_id
WHERE i.stock_quantity / NULLIF(sv.avg_daily_units_sold, 0) <= 7
ORDER BY days_of_stock_remaining ASC;
```

3. Business Scenario: The executive team wants a period-over-period revenue growth report comparing each month to the previous month to gauge business trajectory.

Question: Calculate month-over-month revenue and the percentage growth compared to the previous month.

Return: order_month, monthly_revenue, mom_growth_pct

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date)::date AS order_month,
        SUM(total_amount) AS monthly_revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    order_month,
    monthly_revenue,
    ROUND(
        100.0 * (monthly_revenue - LAG(monthly_revenue) OVER (ORDER BY order_month))
        / NULLIF(LAG(monthly_revenue) OVER (ORDER BY order_month), 0),
        2
    ) AS mom_growth_pct
FROM monthly_revenue
ORDER BY order_month;
```

4. Business Scenario: Logistics leadership wants to measure each courier's SLA compliance rate — how often actual_delivery meets or beats estimated_delivery.

Question: For each courier, calculate the percentage of shipments delivered on or before the estimated_delivery date.

Return: courier, total_shipments, on_time_shipments, sla_compliance_pct

```sql
SELECT
    courier,
    COUNT(*) AS total_shipments,
    SUM(CASE WHEN actual_delivery <= estimated_delivery THEN 1 ELSE 0 END) AS on_time_shipments,
    ROUND(
        100.0 * SUM(CASE WHEN actual_delivery <= estimated_delivery THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS sla_compliance_pct
FROM shipping
WHERE actual_delivery IS NOT NULL
GROUP BY courier
ORDER BY sla_compliance_pct DESC;
```

5. Business Scenario: Partnerships wants a composite seller scorecard blending revenue, customer satisfaction, and delivery reliability to decide on featured-seller status.

Question: Build a seller scorecard combining total revenue, average product rating, and average on-time delivery percentage, ranked from best to worst overall.

Return: seller_id, seller_name, total_revenue, avg_rating, on_time_delivery_pct, overall_rank

```sql
WITH seller_revenue AS (
    SELECT p.seller_id, SUM(oi.subtotal) AS total_revenue
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY p.seller_id
),
seller_avg_rating AS (
    SELECT seller_id, AVG(rating) AS avg_rating
    FROM products
    GROUP BY seller_id
),
seller_delivery AS (
    SELECT
        p.seller_id,
        ROUND(
            100.0 * SUM(CASE WHEN s.actual_delivery <= s.estimated_delivery THEN 1 ELSE 0 END)
            / COUNT(*), 2
        ) AS on_time_delivery_pct
    FROM shipping s
    JOIN orders o ON s.order_id = o.order_id
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE s.actual_delivery IS NOT NULL
    GROUP BY p.seller_id
)
SELECT
    sl.seller_id,
    sl.seller_name,
    COALESCE(sr.total_revenue, 0) AS total_revenue,
    ROUND(sar.avg_rating, 2) AS avg_rating,
    sd.on_time_delivery_pct,
    RANK() OVER (
        ORDER BY COALESCE(sr.total_revenue, 0) DESC,
                 sar.avg_rating DESC,
                 sd.on_time_delivery_pct DESC
    ) AS overall_rank
FROM sellers sl
LEFT JOIN seller_revenue sr ON sl.seller_id = sr.seller_id
LEFT JOIN seller_avg_rating sar ON sl.seller_id = sar.seller_id
LEFT JOIN seller_delivery sd ON sl.seller_id = sd.seller_id
ORDER BY overall_rank;
```

6. Business Scenario: Strategy wants to know which category is growing fastest quarter-over-quarter to prioritize marketing spend.

Question: Calculate quarter-over-quarter revenue growth for each category and identify the fastest-growing category in the most recent quarter.

Return: category_name, quarter, quarterly_revenue, qoq_growth_pct

```sql
WITH quarterly_revenue AS (
    SELECT
        c.category_name,
        DATE_TRUNC('quarter', o.order_date)::date AS quarter,
        SUM(oi.subtotal) AS quarterly_revenue
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    JOIN products p ON oi.product_id = p.product_id
    JOIN category c ON p.category_id = c.category_id
    GROUP BY c.category_name, DATE_TRUNC('quarter', o.order_date)
),
growth AS (
    SELECT
        category_name,
        quarter,
        quarterly_revenue,
        ROUND(
            100.0 * (quarterly_revenue - LAG(quarterly_revenue) OVER (PARTITION BY category_name ORDER BY quarter))
            / NULLIF(LAG(quarterly_revenue) OVER (PARTITION BY category_name ORDER BY quarter), 0),
            2
        ) AS qoq_growth_pct
    FROM quarterly_revenue
)
SELECT *
FROM growth
ORDER BY quarter DESC, qoq_growth_pct DESC NULLS LAST;

-- Fastest-growing category in the most recent quarter only:
-- SELECT * FROM growth WHERE quarter = (SELECT MAX(quarter) FROM growth)
-- ORDER BY qoq_growth_pct DESC NULLS LAST LIMIT 1;
```

7. Business Scenario: The merchandising team wants a single ranked list of "best overall products" blending sales, rating, and review volume rather than ranking on one metric alone.

Question: Build a composite product ranking using a weighted score combining total revenue, average rating, and review count, then rank all products from best to worst.

Return: product_name, total_revenue, avg_rating, review_count, composite_score, overall_rank

```sql
WITH product_metrics AS (
    SELECT
        p.product_id,
        p.product_name,
        COALESCE(SUM(oi.subtotal), 0) AS total_revenue,
        COALESCE(AVG(r.rating), 0) AS avg_rating,
        COUNT(DISTINCT r.review_id) AS review_count
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    LEFT JOIN reviews r ON p.product_id = r.product_id
    GROUP BY p.product_id, p.product_name
),
scored AS (
    SELECT
        *,
        -- weighted composite: 50% revenue (normalized), 30% rating, 20% review volume
        (0.5 * (total_revenue / NULLIF(MAX(total_revenue) OVER (), 0)) * 100)
        + (0.3 * (avg_rating / 5.0) * 100)
        + (0.2 * (review_count / NULLIF(MAX(review_count) OVER (), 0)) * 100) AS composite_score
    FROM product_metrics
)
SELECT
    product_name,
    total_revenue,
    ROUND(avg_rating, 2) AS avg_rating,
    review_count,
    ROUND(composite_score, 2) AS composite_score,
    RANK() OVER (ORDER BY composite_score DESC) AS overall_rank
FROM scored
ORDER BY overall_rank;
```

8. Business Scenario: Finance needs the standard monthly board-report table showing revenue, order volume, and average order value trends over time.

Question: Build a monthly report showing total revenue, total number of orders, and average order value for each month.

Return: order_month, total_revenue, total_orders, avg_order_value

```sql
SELECT
    DATE_TRUNC('month', order_date)::date AS order_month,
    SUM(total_amount) AS total_revenue,
    COUNT(*) AS total_orders,
    ROUND(SUM(total_amount) / COUNT(*), 2) AS avg_order_value
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY order_month;
```

9. Business Scenario: Leadership wants to know what share of total platform revenue comes from repeat customers (2+ orders) versus one-time buyers, to evaluate retention ROI.

Question: Calculate the percentage of total revenue contributed by repeat customers versus one-time customers.

Return: customer_type, total_revenue, pct_of_total_revenue

```sql
WITH customer_orders AS (
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS customer_revenue
    FROM orders
    GROUP BY customer_id
),
classified AS (
    SELECT
        CASE WHEN order_count >= 2 THEN 'Repeat' ELSE 'One-Time' END AS customer_type,
        customer_revenue
    FROM customer_orders
)
SELECT
    customer_type,
    SUM(customer_revenue) AS total_revenue,
    ROUND(100.0 * SUM(customer_revenue) / SUM(SUM(customer_revenue)) OVER (), 2) AS pct_of_total_revenue
FROM classified
GROUP BY customer_type;
```

10. Business Scenario: The finance and growth teams want an estimated Customer Lifetime Value for each customer to guide acquisition spend limits, based on historical order behavior.

Question: Estimate each customer's lifetime value using their total revenue, average order value, order frequency, and customer tenure (registration_date to most recent order_date). Rank customers by estimated CLV.

Return: customer_id, first_name, total_revenue, avg_order_value, order_frequency, tenure_days, estimated_clv, clv_rank

```sql
WITH customer_orders AS (
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS total_revenue,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS last_order_date
    FROM orders
    GROUP BY customer_id
),
clv_base AS (
    SELECT
        c.customer_id,
        c.first_name,
        co.total_revenue,
        ROUND(co.total_revenue / co.order_count, 2) AS avg_order_value,
        ROUND(
            co.order_count / NULLIF(
                EXTRACT(DAY FROM (co.last_order_date - c.registration_date)) / 365.0, 0
            ), 2
        ) AS order_frequency,
        EXTRACT(DAY FROM (co.last_order_date - c.registration_date))::int AS tenure_days
    FROM customers c
    JOIN customer_orders co ON c.customer_id = co.customer_id
)
SELECT
    customer_id,
    first_name,
    total_revenue,
    avg_order_value,
    order_frequency,
    tenure_days,
    ROUND(avg_order_value * order_frequency * (tenure_days / 365.0), 2) AS estimated_clv,
    RANK() OVER (
        ORDER BY avg_order_value * order_frequency * (tenure_days / 365.0) DESC
    ) AS clv_rank
FROM clv_base
ORDER BY clv_rank;
```

