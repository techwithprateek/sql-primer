# Subqueries — Practice Questions

Tables used: `orders`, `order_items`, `order_payments`, `products`, `sellers`, `order_reviews`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Above-average order value.** Find all orders whose total payment value (`SUM(payment_value)` per order) is greater than the overall average order value across the entire dataset.

2. **Sellers with above-average review scores (correlated).** For each product category, find sellers whose average review score is higher than the average review score *for that same category* (i.e. compare each seller to their own category's average, not the global average).

3. **Customers who have ordered (`EXISTS`).** Using `EXISTS`, list all customers (from `customers`) who have placed at least one order — without using a `JOIN`.

4. **Customers who have never ordered (`NOT EXISTS`).** The inverse of Q3 — list customers with zero orders, using `NOT EXISTS` (not `NOT IN`, since `NOT IN` breaks in the presence of `NULL`s).

5. **Scalar subquery in SELECT.** For every order, show the order's total payment value alongside the overall average order value (as a constant column repeated on every row), and the difference between the two — using a scalar subquery in the `SELECT` list, not a window function.
