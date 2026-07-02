# Running Totals — Solutions

### 1. Cumulative revenue by day

```sql
WITH daily_revenue AS (
    SELECT
        DATE(o.order_purchase_timestamp) AS order_date,
        SUM(p.payment_value)             AS revenue
    FROM orders o
    JOIN order_payments p ON p.order_id = o.order_id
    GROUP BY DATE(o.order_purchase_timestamp)
)
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date) AS running_total
FROM daily_revenue
ORDER BY order_date;
```
We first collapse to one row per day with `GROUP BY` (revenue can't be summed twice), then apply the running total on top of that pre-aggregated result — window functions run *after* `GROUP BY` in logical query order, so this two-step CTE approach avoids double-counting.

### 2. Cumulative orders per state

```sql
WITH state_orders AS (
    SELECT
        c.customer_state,
        DATE(o.order_purchase_timestamp) AS order_date,
        COUNT(*)                         AS orders_that_day
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    GROUP BY c.customer_state, DATE(o.order_purchase_timestamp)
)
SELECT
    customer_state,
    order_date,
    orders_that_day,
    SUM(orders_that_day) OVER (
        PARTITION BY customer_state ORDER BY order_date
    ) AS running_orders
FROM state_orders
ORDER BY customer_state, order_date;
```
`PARTITION BY customer_state` is what makes each state's total independent — without it, the running total would blend all states together in purchase-date order.

### 3. Running total with a plateau (date spine)

```sql
WITH date_spine AS (
    SELECT CAST(calendar_date AS DATE) AS calendar_date
    FROM generate_series(
        (SELECT MIN(DATE(order_purchase_timestamp)) FROM orders),
        (SELECT MAX(DATE(order_purchase_timestamp)) FROM orders),
        INTERVAL '1 day'
    ) AS t(calendar_date)
),
daily_revenue AS (
    SELECT
        DATE(o.order_purchase_timestamp) AS order_date,
        SUM(p.payment_value)             AS revenue
    FROM orders o
    JOIN order_payments p ON p.order_id = o.order_id
    GROUP BY DATE(o.order_purchase_timestamp)
)
SELECT
    ds.calendar_date,
    COALESCE(dr.revenue, 0) AS revenue,
    SUM(COALESCE(dr.revenue, 0)) OVER (ORDER BY ds.calendar_date) AS running_total
FROM date_spine ds
LEFT JOIN daily_revenue dr ON dr.order_date = ds.calendar_date
ORDER BY ds.calendar_date;
```
`generate_series` (DuckDB/Postgres) builds one row per calendar day even where no orders exist. The `LEFT JOIN` keeps every spine date; `COALESCE(..., 0)` turns the resulting `NULL` revenue into 0 so the running total carries forward flat instead of breaking.

### 4. Month-to-date (MTD) revenue

```sql
WITH daily_revenue AS (
    SELECT
        DATE(o.order_purchase_timestamp)               AS order_date,
        DATE_TRUNC('month', o.order_purchase_timestamp) AS order_month,
        SUM(p.payment_value)                            AS revenue
    FROM orders o
    JOIN order_payments p ON p.order_id = o.order_id
    GROUP BY 1, 2
)
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (
        PARTITION BY order_month ORDER BY order_date
    ) AS mtd_revenue
FROM daily_revenue
ORDER BY order_date;
```
Same running-total shape as Q1, but `PARTITION BY order_month` resets the accumulator on the 1st of every month — the defining trait of an MTD (vs. all-time) metric.

### 5. Running total of distinct paying customers

```sql
WITH first_purchase AS (
    SELECT
        c.customer_unique_id,
        MIN(DATE(o.order_purchase_timestamp)) AS first_order_date
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    GROUP BY c.customer_unique_id
),
new_customers_by_day AS (
    SELECT first_order_date, COUNT(*) AS new_customers
    FROM first_purchase
    GROUP BY first_order_date
)
SELECT
    first_order_date,
    new_customers,
    SUM(new_customers) OVER (ORDER BY first_order_date) AS cumulative_customers
FROM new_customers_by_day
ORDER BY first_order_date;
```
The trick is computing `MIN(order_date)` per `customer_unique_id` *first* — that collapses each customer to a single "first seen" date — so the running total counts each person exactly once, on the day they first appear.
