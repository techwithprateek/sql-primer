# CTE — Solutions

### 1. Order funnel

```sql
WITH base AS (
    SELECT order_id, order_status FROM orders
),
paid AS (
    SELECT DISTINCT order_id FROM order_payments
),
delivered AS (
    SELECT order_id FROM base WHERE order_status = 'delivered'
),
reviewed AS (
    SELECT DISTINCT order_id FROM order_reviews
)
SELECT
    (SELECT COUNT(*) FROM base)      AS total_orders,
    (SELECT COUNT(*) FROM paid)      AS paid_orders,
    (SELECT COUNT(*) FROM delivered) AS delivered_orders,
    (SELECT COUNT(*) FROM reviewed)  AS reviewed_orders;
```
Each CTE is named for the funnel stage it represents — `base`, `paid`, `delivered`, `reviewed` — so the final `SELECT` reads like a funnel report even though it's just four counts.

### 2. Same funnel, without CTEs (for comparison)

```sql
SELECT
    (SELECT COUNT(*) FROM orders) AS total_orders,
    (SELECT COUNT(DISTINCT order_id) FROM order_payments) AS paid_orders,
    (SELECT COUNT(*) FROM orders WHERE order_status = 'delivered') AS delivered_orders,
    (SELECT COUNT(DISTINCT order_id) FROM order_reviews) AS reviewed_orders;
```
Functionally identical to Q1 — but notice each subquery re-states its own logic inline with no label explaining *why* it represents a funnel stage. On a 3-line query like this it's a minor readability cost; on a 10-step pipeline the CTE version scales far better while this version becomes unreadable.

### 3. Multi-step cohort revenue

```sql
WITH first_order AS (
    SELECT
        c.customer_unique_id,
        MIN(DATE_TRUNC('month', o.order_purchase_timestamp))::date AS cohort_month
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    GROUP BY c.customer_unique_id
),
monthly_revenue AS (
    SELECT
        c.customer_unique_id,
        DATE_TRUNC('month', o.order_purchase_timestamp)::date AS active_month,
        SUM(p.payment_value) AS revenue
    FROM orders o
    JOIN customers c      ON c.customer_id = o.customer_id
    JOIN order_payments p ON p.order_id = o.order_id
    GROUP BY c.customer_unique_id, DATE_TRUNC('month', o.order_purchase_timestamp)
)
SELECT
    fo.cohort_month,
    mr.active_month,
    SUM(mr.revenue) AS cohort_revenue
FROM first_order fo
JOIN monthly_revenue mr ON mr.customer_unique_id = fo.customer_unique_id
GROUP BY fo.cohort_month, mr.active_month
ORDER BY fo.cohort_month, mr.active_month;
```
Three distinct logical steps, each named: `first_order` establishes each customer's cohort, `monthly_revenue` computes their spend per month, and the final `SELECT` joins the two and rolls up to the cohort level. Trying to write this as one flat query without CTEs would require repeating the cohort-month subquery logic multiple times.

### 4. Chained CTEs for top sellers

```sql
WITH seller_revenue AS (
    SELECT oi.seller_id, SUM(oi.price) AS revenue
    FROM order_items oi
    GROUP BY oi.seller_id
),
ranked_sellers AS (
    SELECT
        seller_id,
        revenue,
        RANK() OVER (ORDER BY revenue DESC) AS revenue_rank
    FROM seller_revenue
),
top_10 AS (
    SELECT seller_id, revenue, revenue_rank
    FROM ranked_sellers
    WHERE revenue_rank <= 10
)
SELECT
    t.revenue_rank,
    t.seller_id,
    t.revenue,
    s.seller_city,
    s.seller_state
FROM top_10 t
JOIN sellers s ON s.seller_id = t.seller_id
ORDER BY t.revenue_rank;
```
This is a clean 3-stage pipeline: aggregate (`seller_revenue`) → rank (`ranked_sellers`) → filter (`top_10`) → enrich (final `SELECT` joins back to `sellers`). Each CTE does exactly one job, which is what makes chained CTEs easy to debug — you can run `SELECT * FROM ranked_sellers` on its own to check any intermediate step.

### 5. Recursive CTE — generate a month spine

```sql
WITH RECURSIVE month_spine AS (
    SELECT DATE_TRUNC('month', MIN(order_purchase_timestamp))::date AS month_start
    FROM orders

    UNION ALL

    SELECT (month_start + INTERVAL '1 month')::date
    FROM month_spine
    WHERE month_start < (SELECT DATE_TRUNC('month', MAX(order_purchase_timestamp))::date FROM orders)
)
SELECT month_start FROM month_spine ORDER BY month_start;
```
The anchor (first `SELECT`) computes the earliest month; the recursive step (`UNION ALL` branch) references `month_spine` itself, adding one month each iteration, and stops once it passes the latest month in the data — that `WHERE` clause in the recursive branch is the termination condition without which this would loop forever.
