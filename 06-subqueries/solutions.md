# Subqueries — Solutions

### 1. Above-average order value (non-correlated)

```sql
WITH order_totals AS (
    SELECT order_id, SUM(payment_value) AS order_value
    FROM order_payments
    GROUP BY order_id
)
SELECT *
FROM order_totals
WHERE order_value > (SELECT AVG(order_value) FROM order_totals);
```
The inner `SELECT AVG(order_value) FROM order_totals` is non-correlated — it computes one fixed number, independent of which outer row is being checked, so the database can evaluate it exactly once and reuse it for every comparison.

### 2. Sellers with above-average review scores per category (correlated)

```sql
WITH seller_category_reviews AS (
    SELECT DISTINCT
        oi.seller_id,
        p.product_category_name,
        oi.order_id,
        r.review_score
    FROM order_items oi
    JOIN products p       ON p.product_id = oi.product_id
    JOIN order_reviews r  ON r.order_id = oi.order_id
),
seller_category_scores AS (
    SELECT
        seller_id,
        product_category_name,
        AVG(review_score) AS seller_avg_score
    FROM seller_category_reviews
    GROUP BY seller_id, product_category_name
)
SELECT s.*
FROM seller_category_scores s
WHERE s.seller_avg_score > (
    SELECT AVG(seller_avg_score)
    FROM seller_category_scores s2
    WHERE s2.product_category_name = s.product_category_name
);
```
This is correlated: the inner query's `WHERE s2.product_category_name = s.product_category_name` ties the inner average to whichever category the outer row belongs to, so it effectively recomputes per category. This is the direct SQL translation of "beat your own peer group's average," not the global average.

The extra `seller_category_reviews` step (with `SELECT DISTINCT`) exists for the same reason as in [Aggregations #3](../05-aggregations/solutions.md): joining `order_items` straight to `order_reviews` on `order_id` duplicates a review once per item in that order, so a seller with two same-category items in one order would silently double-weight that review in the average. Deduplicating to one row per `(seller_id, category, order_id)` before aggregating removes that bias.

### 3. Customers who have ordered, using EXISTS

```sql
SELECT c.*
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```
`EXISTS` only checks whether the inner query returns *any* row — it never actually needs to read the `1` it selects, which is why `SELECT 1` (rather than a real column) is the idiomatic way to write it.

### 4. Customers who have never ordered, using NOT EXISTS

```sql
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```
`NOT EXISTS` is preferred over `NOT IN` here because it's `NULL`-safe: if `orders.customer_id` ever contained a `NULL`, `NOT IN (SELECT customer_id FROM orders)` would silently return **zero rows total** (a `NULL` in the list makes every `NOT IN` comparison evaluate to unknown). `NOT EXISTS` has no such trap.

### 5. Scalar subquery in the SELECT list

```sql
WITH order_totals AS (
    SELECT order_id, SUM(payment_value) AS order_value
    FROM order_payments
    GROUP BY order_id
)
SELECT
    order_id,
    order_value,
    (SELECT AVG(order_value) FROM order_totals)                     AS overall_avg_order_value,
    order_value - (SELECT AVG(order_value) FROM order_totals)       AS diff_from_avg
FROM order_totals;
```
A scalar subquery in the `SELECT` list must return exactly one value — here it does, since `AVG()` always collapses to a single number. This produces the same result as `AVG(order_value) OVER ()` (a window function with no `PARTITION BY`), but written as an explicit subquery to show the pattern; in practice, the window-function form is usually preferred since it avoids re-executing the subquery's logic conceptually per row.
