# Deduplication — Practice Questions

Tables used: `order_reviews`, `order_payments`, `geolocation`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Duplicate reviews.** Some orders in `order_reviews` have more than one review row for the same `order_id`. Write a query that returns one row per `order_id`, keeping only the review with the latest `review_creation_date`.

2. **Count before/after.** Write a query that shows, for `order_reviews`, the total row count, the count of distinct `order_id`s, and the difference (i.e. how many duplicate rows exist).

3. **Deduplicate geolocation by zip prefix.** `geolocation` has many rows per `geolocation_zip_code_prefix` (multiple lat/lng samples). Produce one representative row per zip prefix — keep the first row alphabetically by `geolocation_city`.

4. **Safe delete.** Using your answer from Q1, write the corresponding `DELETE` statement that would remove the duplicate review rows, keeping only the latest per order. (Do not run it — just write it.)

5. **Duplicate payments.** Some orders have multiple rows in `order_payments` because a customer split payment across methods (this is legitimate, not a duplicate) — `payment_sequential` distinguishes them. Write a query to confirm this: find any `order_id` + `payment_sequential` combination that appears more than once (a *true* duplicate, unlike the legitimate multi-payment case).
