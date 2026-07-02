# Deduplication — Solutions

### 1. Duplicate reviews — keep the latest

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY order_id
               ORDER BY review_creation_date DESC, review_id DESC
           ) AS rn
    FROM order_reviews
)
SELECT * FROM ranked WHERE rn = 1;
```
`review_id DESC` is a tiebreaker for the (rare) case where two reviews on the same order share the exact same `review_creation_date` — without it, which row survives would be arbitrary.

### 2. Count before/after

```sql
SELECT
    COUNT(*)                      AS total_rows,
    COUNT(DISTINCT order_id)      AS distinct_orders,
    COUNT(*) - COUNT(DISTINCT order_id) AS duplicate_rows
FROM order_reviews;
```
This is the fastest way to *quantify* a dedup problem before writing the fix — if `duplicate_rows` is 0, Q1's `ROW_NUMBER()` logic is unnecessary overhead.

### 3. Deduplicate geolocation by zip prefix

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY geolocation_zip_code_prefix
               ORDER BY geolocation_city ASC
           ) AS rn
    FROM geolocation
)
SELECT
    geolocation_zip_code_prefix,
    geolocation_lat,
    geolocation_lng,
    geolocation_city,
    geolocation_state
FROM ranked
WHERE rn = 1;
```
Same shape as Q1, different partition key and tiebreaker — this is the general-purpose template: change what's inside `PARTITION BY` and `ORDER BY` and the pattern solves almost any "one row per X" requirement.

### 4. Safe delete

```sql
DELETE FROM order_reviews AS o
WHERE EXISTS (
    SELECT 1
    FROM (
        SELECT
            order_id,
            review_id,
            ROW_NUMBER() OVER (
                PARTITION BY order_id
                ORDER BY review_creation_date DESC, review_id DESC
            ) AS rn
        FROM order_reviews
    ) ranked
    WHERE ranked.order_id = o.order_id
      AND ranked.review_id = o.review_id
      AND ranked.rn > 1
);
```
This is the exact inverse of Q1's filter (`rn > 1` instead of `rn = 1`) — always write and eyeball the `SELECT rn = 1` version first; the `DELETE` should only ever remove what that query *didn't* return.

**Why not `WHERE review_id IN (SELECT review_id FROM ranked WHERE rn > 1)`?** It's tempting, but wrong on this dataset: `review_id` on its own is **not globally unique** in `order_reviews` — the same `review_id` can appear attached to several different `order_id`s. An `IN` filter on `review_id` alone would delete every row sharing that id, including legitimate rn = 1 rows that belong to a *different* order. The `EXISTS` version above matches on the full `(order_id, review_id)` pair, so it only ever targets the exact duplicate rows within their own order. On the real dataset, the naive `IN` version deletes 696 rows instead of the correct 551 — a 145-row silent over-deletion that would only surface later as missing reviews for orders that never actually had duplicates.

### 5. True duplicates in order_payments

```sql
SELECT order_id, payment_sequential, COUNT(*) AS row_count
FROM order_payments
GROUP BY order_id, payment_sequential
HAVING COUNT(*) > 1;
```
This is a `GROUP BY` + `HAVING` dedup *check* rather than a fix — it answers "does a true duplicate exist?" without altering data. The key insight is picking the right grain: `order_id` alone isn't the duplicate key here (multiple legitimate payments per order are expected), but `(order_id, payment_sequential)` together should be unique.
