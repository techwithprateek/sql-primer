# Join Patterns — Solutions

### 1. Sellers with no reviews (anti-join across 3 tables)

```sql
SELECT DISTINCT s.seller_id
FROM sellers s
JOIN order_items oi ON oi.seller_id = s.seller_id
LEFT JOIN order_reviews r ON r.order_id = oi.order_id
WHERE r.review_id IS NULL;
```
The first `JOIN` (inner) restricts to sellers who've actually sold something; the second `JOIN` (left) is the anti-join — keeping every order-item row and then filtering for the ones with no matching review. Note the `WHERE` clause only filters on the `LEFT JOIN` side (`r.review_id`), so it never accidentally converts the anti-join back into an inner join.

### 2. Orders missing a review

```sql
SELECT o.order_id
FROM orders o
LEFT JOIN order_reviews r ON r.order_id = o.order_id
WHERE r.review_id IS NULL;
```
The most direct form of the anti-join pattern — one `LEFT JOIN`, one `IS NULL` filter.

### 3. Products never sold

```sql
SELECT p.product_id
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
WHERE oi.order_id IS NULL;
```
Same shape again — the pattern is identical regardless of which two tables you're comparing; only the join key and table names change.

### 4. English category names, with NULL handling

```sql
SELECT
    oi.order_id,
    oi.product_id,
    COALESCE(ct.product_category_name_english, 'unknown') AS category_english
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
LEFT JOIN category_translation ct
    ON ct.product_category_name = p.product_category_name;
```
`LEFT JOIN` on the translation table is required because some `product_category_name` values are `NULL` or don't have a match in the translation lookup — an `INNER JOIN` there would silently drop those order items entirely. `COALESCE` then converts the resulting `NULL` into a readable `'unknown'` label.

### 5. Full outer comparison + referential integrity check

```sql
SELECT
    COALESCE(s.seller_id, oi.seller_id) AS seller_id,
    CASE
        WHEN s.seller_id IS NULL THEN 'in order_items only (orphan seller_id)'
        WHEN oi.seller_id IS NULL THEN 'in sellers only (never sold anything)'
        ELSE 'matched'
    END AS match_status
FROM sellers s
FULL JOIN order_items oi ON oi.seller_id = s.seller_id
WHERE s.seller_id IS NULL OR oi.seller_id IS NULL
GROUP BY 1, 2;
```
A `FULL JOIN` keeps unmatched rows from *both* sides at once, which is exactly what a referential integrity check needs — an `INNER JOIN` would hide both failure modes, and a single `LEFT JOIN` would only catch one direction. The `CASE` on which side is `NULL` tells you *which* problem you're looking at.
