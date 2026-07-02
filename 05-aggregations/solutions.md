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
SELECT
    oi.seller_id,
    AVG(r.review_score)  AS avg_review_score,
    COUNT(r.review_id)   AS review_count
FROM order_items oi
JOIN order_reviews r ON r.order_id = oi.order_id
GROUP BY oi.seller_id
HAVING COUNT(r.review_id) >= 20
ORDER BY avg_review_score DESC;
```
Note the join is on `order_id`, not `seller_id` directly — reviews are attached to orders, not sellers, so you have to go through `order_items` to connect the two. If an order has multiple sellers (multi-seller cart), that order's single review gets attributed to each seller involved — a known limitation of this dataset worth calling out in an interview.

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
SELECT
    oi.seller_id,
    COUNT(DISTINCT o.order_id) AS total_orders,
    AVG(DATE(o.order_delivered_customer_date) - DATE(o.order_purchase_timestamp)) AS avg_delivery_days,
    100.0 * COUNT(DISTINCT CASE
        WHEN o.order_delivered_customer_date > o.order_estimated_delivery_date
        THEN o.order_id
    END) / COUNT(DISTINCT o.order_id) AS pct_late_deliveries
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.order_delivered_customer_date IS NOT NULL
GROUP BY oi.seller_id
HAVING COUNT(DISTINCT o.order_id) >= 50
ORDER BY pct_late_deliveries DESC;
```
Three techniques combined: a `WHERE` clause up front removes undelivered orders (avoids `NULL` dates poisoning the date-difference average), a plain `AVG()` handles the delivery-time metric, and a conditional `COUNT(DISTINCT CASE WHEN ...)` divided by the total gives a percentage — all computed together instead of chaining several separate queries.
