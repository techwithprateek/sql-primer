# 2. Deduplication

## Concept

Real data is rarely clean — the same logical record can appear more than once because of retries, upstream bugs, re-imports, or legitimately repeated events that need collapsing to "one per key." Deduplication means defining what makes two rows "the same" (the key) and a rule for which copy to keep (the tiebreaker).

`SELECT DISTINCT` only helps when rows are *fully* identical. The moment two duplicate rows differ in even one column (e.g. a slightly later `updated_at`), you need something smarter: rank the duplicates within each key and keep rank 1.

## Syntax

```sql
ROW_NUMBER() OVER (
    PARTITION BY <duplicate_key_columns>
    ORDER BY <tiebreaker_column>
)
```

`PARTITION BY` defines what counts as "the same record"; `ORDER BY` decides which copy wins (usually "most recent" or "smallest id"). Wrap it in a CTE and filter `WHERE rn = 1` to keep exactly one row per key.

## Generic Example

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id, order_date
               ORDER BY order_id
           ) AS rn
    FROM orders
)
SELECT * FROM ranked WHERE rn = 1;
```

This keeps the first (lowest `order_id`) row per `(customer_id, order_date)` pair and discards the rest.

## Common Gotchas

- **Picking the wrong key** — deduplicating on too few columns silently deletes rows that were actually distinct; too many columns and true duplicates slip through.
- **`ROW_NUMBER()` vs `RANK()`/`DENSE_RANK()`** — for deduplication you almost always want `ROW_NUMBER()`, since it never ties (each row gets a unique number), guaranteeing exactly one `rn = 1` per partition.
- **No stable tiebreaker** — if two duplicate rows are identical even in the `ORDER BY` column, which one survives is arbitrary. Add the primary key as a final tiebreaker (`ORDER BY updated_at DESC, id DESC`) for reproducibility.
- **`DELETE` vs `SELECT`** — the pattern above is written as a `SELECT` (safe, non-destructive). To actually remove duplicates in place, wrap the same logic in `DELETE FROM t WHERE id IN (SELECT id FROM ranked WHERE rn > 1)` — and always run the `SELECT` version first to check what you're about to delete.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
