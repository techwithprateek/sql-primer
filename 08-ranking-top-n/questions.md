# Ranking / Top-N per Group — Practice Questions

Tables used: `orders`, `order_items`, `products`, `customers`, `category_translation`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Top 5 sellers by revenue per state.** For each `seller_state`, find the top 5 sellers by total revenue (`SUM(price)` from `order_items`).

2. **Best-selling product category per month.** For each calendar month, find the single product category (English name) with the highest total revenue.

3. **Most recent order per customer.** For every customer (`customer_unique_id`) with more than one order, return only their single most recent order.

4. **Handling ties — second-highest order value.** Find each customer's second-highest single order value. If a customer has two orders tied for the highest value, their "second highest" should be the *next distinct* value below the tie (i.e. use `DENSE_RANK()`, not `ROW_NUMBER()`).

5. **Top 3 categories by order count, with ties shown.** For each `customer_state`, find the top 3 product categories by number of orders — but if multiple categories tie for 3rd place, show all of them (i.e. the result may have more than 3 rows for a state).
