# 5. Aggregations

## Concept

Aggregation collapses many rows into one summary row per group — total revenue per region, average order value per customer, count of returns per product. It's the backbone of almost every dashboard metric, and it's built from three pieces working together: a grouping key (`GROUP BY`), an aggregate function (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`), and an optional filter on the *aggregated* result (`HAVING`).

The one idea that trips people up: `WHERE` filters rows **before** grouping; `HAVING` filters groups **after** aggregation. You cannot write `WHERE SUM(revenue) > 1000` — the sum doesn't exist yet at the point `WHERE` runs.

## Syntax

```sql
SELECT
    <group_column>,
    <AGG_FUNC>(<column>) AS <alias>
FROM <table>
WHERE <row-level filter>
GROUP BY <group_column>
HAVING <group-level filter>;
```

**Conditional aggregation** — using `CASE WHEN` inside an aggregate function to compute multiple conditional metrics in a single pass, instead of one query per condition:

```sql
COUNT(CASE WHEN status = 'returned' THEN 1 END) AS returns
```

## Generic Example

```sql
SELECT
    region,
    SUM(revenue)                                        AS total_revenue,
    AVG(order_value)                                     AS avg_order_value,
    COUNT(CASE WHEN status = 'returned' THEN 1 END)      AS returns
FROM orders
GROUP BY region
HAVING SUM(revenue) > 100000;
```

This returns one row per region, but only for regions whose total revenue exceeds 100,000 — `WHERE` couldn't express that last condition because it needs the aggregated `SUM()` to already exist.

## Common Gotchas

- **Non-aggregated columns not in `GROUP BY`** — every column in the `SELECT` list must either be in `GROUP BY` or wrapped in an aggregate function. Most databases will error on this; a few (like older MySQL configs) silently return an arbitrary row, which is worse.
- **`COUNT(*)` vs `COUNT(column)`** — `COUNT(*)` counts rows regardless of `NULL`s; `COUNT(column)` skips rows where that column is `NULL`. Mixing these up silently changes your denominator.
- **`AVG` after a fan-out join** — if you joined `orders` to `order_items` before aggregating, `AVG(order_total)` will be wrong because the order-level value got duplicated once per item. Aggregate to the right grain first (see [Join Patterns](../04-join-patterns/)).
- **Forgetting `HAVING` runs after `GROUP BY`** — trying to filter on an aggregate in `WHERE` throws an error (or is silently rejected) because `WHERE` operates on raw rows, before any grouping has happened.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
