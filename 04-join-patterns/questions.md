# Join Patterns — Practice Questions

Tables used: `orders`, `order_items`, `order_reviews`, `sellers`, `products`, `category_translation`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Sellers with no reviews.** Find every seller who has sold at least one item but has never received a single review on any of their orders (anti-join).

2. **Orders missing a review.** List every `order_id` that has no matching row in `order_reviews` at all.

3. **Products never sold.** Find every `product_id` in `products` that has never appeared in `order_items` (i.e. it exists in the catalog but was never ordered).

4. **English category names.** Join `order_items` → `products` → `category_translation` to produce a result set with the English category name for every order item. Handle products where `product_category_name` is missing (`NULL`) gracefully — label them `'unknown'`.

5. **Full outer comparison.** Using a `FULL JOIN` between `sellers` and `order_items`, find (a) sellers who exist but have never sold anything, and (b) any `seller_id` present in `order_items` but missing from `sellers` (a referential integrity check) — in one query, with a column indicating which side each row came from.
