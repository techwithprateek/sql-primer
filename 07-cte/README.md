# 7. CTE (Common Table Expressions)

## Concept

A CTE (`WITH` clause) names an intermediate result so you can build a complex query as a sequence of readable, logical steps instead of one deeply nested block. It doesn't do anything a subquery can't do — the value is entirely in **readability and reuse**: give each step a name that says what it represents, and the whole query reads top-to-bottom like a recipe instead of inside-out like a subquery.

Two CTE variants worth knowing:
- **Chained CTEs** — each CTE can reference the ones defined before it, letting you build up a multi-step pipeline (raw data → filtered → aggregated → ranked).
- **Recursive CTEs** — a CTE that references *itself*, used for hierarchical or graph-like data (org charts, category trees, bill-of-materials explosions). Not common in analyst interviews, but a strong signal of SQL fluency when it comes up.

## Syntax

**Chained CTEs:**
```sql
WITH step_one AS (
    SELECT ...
),
step_two AS (
    SELECT ... FROM step_one WHERE ...
)
SELECT ... FROM step_two;
```

**Recursive CTE:**
```sql
WITH RECURSIVE hierarchy AS (
    -- anchor: the base case (e.g. top-level rows with no parent)
    SELECT id, parent_id, 1 AS level FROM nodes WHERE parent_id IS NULL

    UNION ALL

    -- recursive step: join back to the CTE itself
    SELECT n.id, n.parent_id, h.level + 1
    FROM nodes n
    JOIN hierarchy h ON n.parent_id = h.id
)
SELECT * FROM hierarchy;
```

## Generic Example

```sql
WITH funnel AS (
    SELECT order_id, order_date FROM orders
),
paid AS (
    SELECT order_id FROM payments WHERE status = 'paid'
),
delivered AS (
    SELECT order_id FROM shipments WHERE status = 'delivered'
)
SELECT
    f.order_id,
    CASE WHEN p.order_id IS NOT NULL THEN 1 ELSE 0 END AS is_paid,
    CASE WHEN d.order_id IS NOT NULL THEN 1 ELSE 0 END AS is_delivered
FROM funnel f
LEFT JOIN paid p      ON f.order_id = p.order_id
LEFT JOIN delivered d ON f.order_id = d.order_id;
```

Each CTE isolates one stage of the funnel. Even without comments, the query is self-documenting — `funnel`, `paid`, and `delivered` describe exactly what each block represents.

## Common Gotchas

- **CTEs aren't automatically indexed or materialized for performance** — in most databases (Postgres, DuckDB), a CTE is inlined into the query plan like a subquery, not pre-computed and cached. Don't assume referencing the same CTE twice avoids recomputation (some engines materialize once per reference, some don't — check your engine's behavior if performance matters).
- **Scope is one query only** — a CTE only exists for the duration of the single statement it's attached to. It's not a temp table you can reuse across multiple queries.
- **Recursive CTEs need a termination condition** — without one, a self-referencing recursive CTE can loop indefinitely. Always make sure the recursive step's join condition eventually stops producing new rows (e.g. `parent_id` chains upward to a row with `NULL` parent).
- **Over-chaining** — CTEs are a readability tool, not a performance one. Ten nested CTEs "for the sake of it" can be just as hard to follow as one giant subquery; name and split them because each step is a genuinely distinct logical stage, not by default.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
