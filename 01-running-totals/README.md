# 1. Running Totals

## Concept

A running total (a.k.a. cumulative sum) is a value that accumulates as you move through rows in a defined order — each row's total includes everything before it. The classic example is a bank balance: every transaction shows the balance *after* it, not just the transaction amount.

You can't do this with `GROUP BY` alone, because `GROUP BY` collapses rows — it doesn't let you see "everything up to this row." That's what **window functions** are for: they let a row see other rows (an ordered "window" around it) without collapsing the result set.

## Syntax

```sql
SUM(<value>) OVER (
    [PARTITION BY <group_column>]
    ORDER BY <ordering_column>
)
```

- **`PARTITION BY`** *(optional)* — restart the running total for each group (e.g. per customer, per region). Omit it to run the total across the whole table.
- **`ORDER BY`** *(required)* — defines the sequence the total accumulates in. Almost always a date or timestamp.

## Generic Example

```sql
SELECT
    order_date,
    daily_sales,
    SUM(daily_sales) OVER (ORDER BY order_date) AS running_total
FROM sales;
```

| order_date | daily_sales | running_total |
|---|---|---|
| 2026-01-01 | 100 | 100 |
| 2026-01-02 | 50  | 150 |
| 2026-01-03 | 75  | 225 |

Add `PARTITION BY region` and each region gets its own independent running total, restarting from zero.

## Common Gotchas

- **Forgetting `ORDER BY`** — without it, SQL doesn't know what "running" means and may sum the whole partition for every row.
- **Ties in the `ORDER BY` column** — by default, rows with the same value in the `ORDER BY` column (e.g. two orders on the same timestamp) are summed together in one step (`RANGE` framing) rather than accumulating one at a time (`ROWS` framing). Use `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` if you need strict row-by-row accumulation regardless of ties.
- **Confusing this with a moving average** — a running total always includes everything from the start; a moving average uses a fixed-size window (e.g. last 7 rows) via `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
