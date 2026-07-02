# Aggregations — Practice Questions

Tables used: `orders`, `order_items`, `order_payments`, `order_reviews`, `customers`, `sellers`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Revenue by state.** Total revenue (`order_payments.payment_value`) grouped by `customer_state`, sorted highest to lowest.

2. **High-volume states only.** Same as Q1, but only show states with more than 1,000 distinct orders (use `HAVING`).

3. **Average review score per seller.** For each seller, compute the average `review_score` across all their orders' reviews, but only include sellers with at least 20 reviews.

4. **Conditional aggregation — payment method mix.** For each `customer_state`, in a single query, show: total orders, count of orders paid by `credit_card`, count paid by `boleto`, and count paid by `voucher` (three separate conditional counts side by side).

5. **Delivery performance by seller.** For each seller, compute: total orders shipped, average days between `order_purchase_timestamp` and `order_delivered_customer_date`, and the percentage of orders delivered after `order_estimated_delivery_date` (late deliveries). Only include sellers with 50+ orders.
