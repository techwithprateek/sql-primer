# Ranking / Top-N per Group — Solutions

### 1. Top 5 sellers by revenue per state

```sql
WITH seller_revenue AS (
    SELECT
        s.seller_state,
        s.seller_id,
        SUM(oi.price) AS revenue
    FROM order_items oi
    JOIN sellers s ON s.seller_id = oi.seller_id
    GROUP BY s.seller_state, s.seller_id
),
ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY seller_state ORDER BY revenue DESC) AS rnk
    FROM seller_revenue
)
SELECT * FROM ranked WHERE rnk <= 5
ORDER BY seller_state, rnk;
```
`ROW_NUMBER()` is the right choice here (over `RANK()`) because "top 5 sellers" is a fixed headcount request — if two sellers tied for 5th, the business question ("give me 5 sellers") implies picking exactly one of them, not returning 6.

### 2. Best-selling category per month

```sql
WITH monthly_category_revenue AS (
    SELECT
        DATE_TRUNC('month', o.order_purchase_timestamp)::date AS order_month,
        COALESCE(ct.product_category_name_english, 'unknown') AS category,
        SUM(oi.price) AS revenue
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
    JOIN products p ON p.product_id = oi.product_id
    LEFT JOIN category_translation ct ON ct.product_category_name = p.product_category_name
    GROUP BY 1, 2
),
ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY order_month ORDER BY revenue DESC) AS rnk
    FROM monthly_category_revenue
)
SELECT order_month, category, revenue
FROM ranked
WHERE rnk = 1
ORDER BY order_month;
```
`PARTITION BY order_month` with `rnk = 1` is the "top 1 per group" special case of the general top-N pattern — same shape as Q1, just filtering to a single winner instead of five.

### 3. Most recent order per customer

```sql
WITH customer_orders AS (
    SELECT
        c.customer_unique_id,
        o.order_id,
        o.order_purchase_timestamp,
        COUNT(*) OVER (PARTITION BY c.customer_unique_id) AS total_orders
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
),
ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_unique_id ORDER BY order_purchase_timestamp DESC
        ) AS rnk
    FROM customer_orders
)
SELECT customer_unique_id, order_id, order_purchase_timestamp
FROM ranked
WHERE rnk = 1 AND total_orders > 1
ORDER BY order_purchase_timestamp DESC;
```
Two window functions stacked: `COUNT(*) OVER (PARTITION BY ...)` (no `ORDER BY`) counts each customer's total orders without collapsing rows, letting us filter to repeat customers only; `ROW_NUMBER()` (with `ORDER BY ... DESC`) then picks their latest order.

### 4. Second-highest order value per customer, tie-safe

```sql
WITH order_totals AS (
    SELECT
        c.customer_unique_id,
        o.order_id,
        SUM(p.payment_value) AS order_value
    FROM orders o
    JOIN customers c      ON c.customer_id = o.customer_id
    JOIN order_payments p ON p.order_id = o.order_id
    GROUP BY c.customer_unique_id, o.order_id
),
ranked AS (
    SELECT
        *,
        DENSE_RANK() OVER (PARTITION BY customer_unique_id ORDER BY order_value DESC) AS rnk
    FROM order_totals
)
SELECT customer_unique_id, order_id, order_value
FROM ranked
WHERE rnk = 2;
```
`DENSE_RANK()` is essential here, not a stylistic choice: if a customer's top two orders are tied at, say, $500, `ROW_NUMBER()` would arbitrarily assign one of them rank 2 — incorrectly treating a tied value as "second highest." `DENSE_RANK()` gives both $500 orders rank 1, so rank 2 correctly lands on the next *distinct* lower value.

### 5. Top 3 categories per state, ties included

```sql
WITH state_category_orders AS (
    SELECT
        c.customer_state,
        COALESCE(ct.product_category_name_english, 'unknown') AS category,
        COUNT(DISTINCT o.order_id) AS order_count
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON p.product_id = oi.product_id
    LEFT JOIN category_translation ct ON ct.product_category_name = p.product_category_name
    GROUP BY 1, 2
),
ranked AS (
    SELECT
        *,
        RANK() OVER (PARTITION BY customer_state ORDER BY order_count DESC) AS rnk
    FROM state_category_orders
)
SELECT * FROM ranked WHERE rnk <= 3
ORDER BY customer_state, rnk;
```
This time `RANK()` is the right choice (the opposite call from Q1): the question explicitly asks to *show* ties rather than force a fixed count, and `RANK()` naturally lets multiple categories share rank 3 — all of them satisfy `rnk <= 3` and appear in the output, exactly matching the business ask.
