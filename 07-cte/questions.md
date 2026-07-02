# CTE — Practice Questions

Tables used: `orders`, `order_items`, `order_payments`, `order_reviews`, `customers`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Order funnel.** Using chained CTEs, build a funnel showing, for every order: whether it was paid (exists in `order_payments`), whether it was delivered (`order_status = 'delivered'`), and whether it was reviewed (exists in `order_reviews`). Return counts at each funnel stage (total orders → paid → delivered → reviewed).

2. **Rewrite without CTEs.** Take your answer to Q1 and rewrite it as a single query using only nested subqueries (no `WITH`). Compare readability — this is the point of the exercise, not a query you'd actually ship.

3. **Multi-step cohort revenue.** Using CTEs, compute: (a) each customer's first order month (their "cohort"), (b) their total revenue per calendar month after that, (c) a final result showing revenue by cohort month × active month (a basic cohort revenue table).

4. **Chained CTEs for top sellers.** Build a 3-step CTE chain: step 1 computes revenue per seller, step 2 ranks sellers by revenue, step 3 filters to only the top 10 and joins back to `sellers` for their city/state.

5. **(Stretch) Recursive CTE.** The Olist dataset doesn't have a natural hierarchy, so instead: write a recursive CTE that generates a series of all calendar months between the earliest and latest `order_purchase_timestamp` in the dataset (i.e. build your own date-spine using recursion instead of `generate_series`).
