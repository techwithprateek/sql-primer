# Gaps & Islands — Solutions

### 1. Calendar gaps (days with zero orders)

```sql
WITH order_dates AS (
    SELECT DISTINCT DATE(order_purchase_timestamp) AS order_date
    FROM orders
),
numbered AS (
    SELECT
        order_date,
        order_date - (ROW_NUMBER() OVER (ORDER BY order_date) * INTERVAL '1 day') AS grp
    FROM order_dates
),
islands AS (
    SELECT MIN(order_date) AS island_start, MAX(order_date) AS island_end
    FROM numbered
    GROUP BY grp
)
SELECT
    island_end + INTERVAL '1 day'                                        AS gap_start,
    LEAD(island_start) OVER (ORDER BY island_start) - INTERVAL '1 day'    AS gap_end,
    (LEAD(island_start) OVER (ORDER BY island_start) - island_end) - 1    AS gap_length_days
FROM islands
QUALIFY gap_length_days > 0
ORDER BY gap_start;
```
This is the two-step "islands, then gaps" approach from the README: first collapse consecutive order-dates into islands (runs of active days), then use `LEAD()` to look at the *next* island's start and infer the gap between islands. (No `QUALIFY` support? Wrap the final `SELECT` in a CTE and filter with a `WHERE` clause on the outer query instead.)

### 2. Seller active-month streaks

```sql
WITH seller_months AS (
    SELECT DISTINCT
        oi.seller_id,
        DATE_TRUNC('month', o.order_purchase_timestamp)::date AS active_month
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
),
numbered AS (
    SELECT
        seller_id,
        active_month,
        active_month - (ROW_NUMBER() OVER (
            PARTITION BY seller_id ORDER BY active_month
        ) * INTERVAL '1 month') AS grp
    FROM seller_months
),
streaks AS (
    SELECT
        seller_id,
        MIN(active_month) AS streak_start,
        MAX(active_month) AS streak_end,
        COUNT(*)          AS streak_length_months
    FROM numbered
    GROUP BY seller_id, grp
)
SELECT *
FROM streaks
QUALIFY ROW_NUMBER() OVER (PARTITION BY seller_id ORDER BY streak_length_months DESC) = 1
ORDER BY streak_length_months DESC;
```
`DATE_TRUNC('month', ...)` normalizes every order to the 1st of its month so consecutive months are exactly 1 month apart, matching the interval subtracted in `numbered`. The final `QUALIFY`/`ROW_NUMBER()` keeps only each seller's single *longest* streak (a seller can have multiple streaks; we want the best one).

### 3. Seller inactivity gaps ≥ 60 days

```sql
WITH seller_sales AS (
    SELECT DISTINCT oi.seller_id, DATE(o.order_purchase_timestamp) AS sale_date
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
),
with_prev AS (
    SELECT
        seller_id,
        sale_date,
        LAG(sale_date) OVER (PARTITION BY seller_id ORDER BY sale_date) AS prev_sale_date
    FROM seller_sales
)
SELECT
    seller_id,
    prev_sale_date + INTERVAL '1 day' AS gap_start,
    sale_date - prev_sale_date        AS gap_length_days
FROM with_prev
WHERE sale_date - prev_sale_date >= 60
ORDER BY gap_length_days DESC;
```
This uses the `LAG()` variant of the pattern instead of the row-number-offset variant — for a simple "flag rows where the jump from the previous row is too big" question, comparing directly to `LAG()` is more direct than building full islands first.

### 4. Repeat-customer longest streak

```sql
WITH customer_order_count AS (
    SELECT c.customer_unique_id, COUNT(*) AS order_count
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    GROUP BY c.customer_unique_id
    HAVING COUNT(*) >= 3
),
customer_months AS (
    SELECT DISTINCT
        c.customer_unique_id,
        DATE_TRUNC('month', o.order_purchase_timestamp)::date AS order_month
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    WHERE c.customer_unique_id IN (SELECT customer_unique_id FROM customer_order_count)
),
numbered AS (
    SELECT
        customer_unique_id,
        order_month,
        order_month - (ROW_NUMBER() OVER (
            PARTITION BY customer_unique_id ORDER BY order_month
        ) * INTERVAL '1 month') AS grp
    FROM customer_months
),
streaks AS (
    SELECT
        customer_unique_id,
        MIN(order_month) AS streak_start,
        MAX(order_month) AS streak_end,
        COUNT(*)          AS streak_length_months
    FROM numbered
    GROUP BY customer_unique_id, grp
)
SELECT *
FROM streaks
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_unique_id ORDER BY streak_length_months DESC) = 1
ORDER BY streak_length_months DESC;
```
The `customer_order_count` CTE pre-filters to customers with 3+ orders *before* the expensive island logic runs — filtering early keeps the window function working over a much smaller set.

### 5. Order-volume surge islands (above-average consecutive days)

```sql
WITH daily_counts AS (
    SELECT DATE(order_purchase_timestamp) AS order_date, COUNT(*) AS order_count
    FROM orders
    GROUP BY DATE(order_purchase_timestamp)
),
avg_daily AS (
    SELECT AVG(order_count) AS avg_orders FROM daily_counts
),
above_avg_days AS (
    SELECT order_date, order_count
    FROM daily_counts, avg_daily
    WHERE order_count > avg_daily.avg_orders
),
numbered AS (
    SELECT
        order_date,
        order_count,
        order_date - (ROW_NUMBER() OVER (ORDER BY order_date) * INTERVAL '1 day') AS grp
    FROM above_avg_days
)
SELECT
    MIN(order_date)      AS surge_start,
    MAX(order_date)      AS surge_end,
    COUNT(*)             AS surge_length_days,
    SUM(order_count)     AS total_orders_in_surge
FROM numbered
GROUP BY grp
ORDER BY surge_start;
```
Notice the two filters happen in sequence: first reduce to "days above average" (a `WHERE`), *then* apply the islands trick only to that filtered set — the row numbering in `numbered` only makes sense once the non-surge days have already been removed, since islands are defined purely by adjacency within the filtered rows.
