# Running Totals — Practice Questions

Tables used: `orders`, `order_items`, `order_payments`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Cumulative revenue by day.** For every day in the dataset, show the day's revenue (`sum of order_payments.payment_value` for orders placed that day) and the running total of revenue up to and including that day.

2. **Cumulative orders per state.** For each `customer_state`, show a running count of orders over time, restarting at zero for each state.

3. **Running total with a plateau.** On days with zero orders, the running total should carry forward the previous total rather than showing a gap. Write a query that produces one row per calendar day (including days with no orders) with the correct running total. *(Hint: you'll need a date spine — a generated series of every date in range — left-joined to your daily revenue.)*

4. **Month-to-date (MTD) revenue.** Show cumulative revenue that resets at the start of each month, instead of accumulating across the entire dataset.

5. **Running total of paying customers.** Show a running count of *distinct* customers (`customer_unique_id`) who have placed at least one order, ordered by their first purchase date. Each customer should only be counted once, on the day of their first order.
