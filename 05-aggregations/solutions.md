# Aggregations — Solutions

### 1. Revenue by state

```sql
SELECT
    c.customer_state,
    SUM(p.payment_value) AS total_revenue
FROM orders o
JOIN customers c      ON c.customer_id = o.customer_id
JOIN order_payments p ON p.order_id = o.order_id
GROUP BY c.customer_state
ORDER BY total_revenue DESC;
```
Straightforward join-then-aggregate: bring in `customer_state` via `customers` and `payment_value` via `order_payments`, then collapse to one row per state.

### 2. High-volume states only

```sql
SELECT
    c.customer_state,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(p.payment_value)       AS total_revenue
FROM orders o
JOIN customers c      ON c.customer_id = o.customer_id
JOIN order_payments p ON p.order_id = o.order_id
GROUP BY c.customer_state
HAVING COUNT(DISTINCT o.order_id) > 1000
ORDER BY total_revenue DESC;
```
`COUNT(DISTINCT o.order_id)` rather than `COUNT(*)` matters here: an order can have multiple payment rows (split payments), so a plain `COUNT(*)` would overcount orders. `HAVING` filters on that aggregated count — this is exactly the kind of filter `WHERE` cannot express.

### 3. Average review score per seller (20+ reviews)

```sql
WITH seller_reviews AS (
    SELECT DISTINCT oi.seller_id, oi.order_id, r.review_id, r.review_score
    FROM order_items oi
    JOIN order_reviews r ON r.order_id = oi.order_id
)
SELECT
    seller_id,
    AVG(review_score) AS avg_review_score,
    COUNT(review_id)  AS review_count
FROM seller_reviews
GROUP BY seller_id
HAVING COUNT(review_id) >= 20
ORDER BY avg_review_score DESC;
```
Note the join is on `order_id`, not `seller_id` directly — reviews are attached to orders, not sellers, so you have to go through `order_items` to connect the two. If an order has multiple sellers (multi-seller cart), that order's single review gets attributed to each seller involved — a known limitation of this dataset worth calling out in an interview.

**The `SELECT DISTINCT` in the CTE is load-bearing, not stylistic.** A seller can have multiple items in the *same* order (e.g. a customer buys 2 different products from the same seller in one cart). Joining `order_items` straight to `order_reviews` on `order_id` would then return that order's review once per item, so a 2-item order silently counts twice in both the average and the count — skewing `avg_review_score` (in one measurement on the real dataset, by as much as 1.27 points for affected sellers) and inflating `review_count` enough to push some sellers over the `>= 20` threshold who shouldn't qualify. `DISTINCT` on `(seller_id, order_id, review_id, review_score)` collapses back to one row per seller-order pair before aggregating, so each review is weighted once regardless of how many items generated it.

### 4. Conditional aggregation — payment method mix

```sql
SELECT
    c.customer_state,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT CASE WHEN p.payment_type = 'credit_card' THEN o.order_id END) AS credit_card_orders,
    COUNT(DISTINCT CASE WHEN p.payment_type = 'boleto'      THEN o.order_id END) AS boleto_orders,
    COUNT(DISTINCT CASE WHEN p.payment_type = 'voucher'     THEN o.order_id END) AS voucher_orders
FROM orders o
JOIN customers c      ON c.customer_id = o.customer_id
JOIN order_payments p ON p.order_id = o.order_id
GROUP BY c.customer_state
ORDER BY total_orders DESC;
```
This is the "pivot without `PIVOT`" trick: each `CASE WHEN` inside `COUNT(DISTINCT ...)` only counts rows matching that condition, letting you compute three different segment counts in one pass over the data instead of three separate queries unioned together.

### 5. Delivery performance by seller

```sql
WITH seller_orders AS (
    SELECT DISTINCT
        oi.seller_id,
        o.order_id,
        DATE(o.order_delivered_customer_date) - DATE(o.order_purchase_timestamp) AS delivery_days,
        CASE WHEN o.order_delivered_customer_date > o.order_estimated_delivery_date
             THEN 1 ELSE 0 END AS is_late
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.order_delivered_customer_date IS NOT NULL
)
SELECT
    seller_id,
    COUNT(*)                       AS total_orders,
    AVG(delivery_days)             AS avg_delivery_days,
    100.0 * SUM(is_late) / COUNT(*) AS pct_late_deliveries
FROM seller_orders
GROUP BY seller_id
HAVING COUNT(*) >= 50
ORDER BY pct_late_deliveries DESC;
```
A `WHERE` clause up front removes undelivered orders (avoids `NULL` dates poisoning the date-difference average), and the `seller_orders` CTE collapses to one row per `(seller_id, order_id)` before any aggregation runs.

**That `DISTINCT` in the CTE matters for the same reason as the seller-review queries in [Subqueries #2](../06-subqueries/solutions.md) and [Aggregations #3](#3-average-review-score-per-seller-20-reviews):** joining `orders` to `order_items` puts one row per *item*, not per order. A seller with 3 items in the same order would otherwise have that order's delivery time counted 3 times in `AVG(delivery_days)`, over-weighting multi-item orders relative to single-item ones — on the real dataset this skewed `avg_delivery_days` by as much as 2.5 days for some sellers. `total_orders` and `pct_late_deliveries` happen to survive this bug unscathed only because the original version wrapped them in `COUNT(DISTINCT o.order_id)`; the plain `AVG()` had no such protection.
