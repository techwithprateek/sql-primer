# 8. Ranking / Top-N per Group

## Concept

"Top N per group" — top 3 highest-paid employees per department, best-selling product per category, most recent order per customer — is one of the most common real-world query shapes, and one plain `GROUP BY` cannot answer, because `GROUP BY` can only return an *aggregate* per group (a max, a count), not the *full row* that produced it. Ranking window functions solve this by numbering rows within each group without collapsing them, so you can then filter to "rank <= N."

Three ranking functions, and the difference between them only matters when there are ties:

| Function | Behavior on ties | Example (scores 90, 90, 80) |
|---|---|---|
| `ROW_NUMBER()` | Always unique, arbitrary tiebreak | 1, 2, 3 |
| `RANK()` | Ties share a rank; next rank skips | 1, 1, 3 |
| `DENSE_RANK()` | Ties share a rank; next rank doesn't skip | 1, 1, 2 |

## Syntax

```sql
SELECT *
FROM (
    SELECT
        *,
        RANK() OVER (PARTITION BY <group_column> ORDER BY <ranking_column> DESC) AS rnk
    FROM <table>
) ranked
WHERE rnk <= N;
```

`PARTITION BY` defines the group each row is ranked *within*; `ORDER BY` defines what "best" means (usually `DESC` for "top").

## Generic Example

```sql
WITH ranked AS (
    SELECT
        department_id,
        employee_id,
        salary,
        RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT * FROM ranked WHERE rnk <= 3;
```

For each department, this ranks employees by salary independently, then keeps only the top 3 rows per department.

## Common Gotchas

- **Filtering rank in the same `SELECT` as the window function fails** — you can't write `WHERE rnk <= 3` in the same query that defines `rnk`, because `WHERE` runs before window functions are computed. Always wrap the ranking in a CTE or subquery first, then filter in the outer query.
- **Wrong choice between `RANK()`/`ROW_NUMBER()`/`DENSE_RANK()` for "top N"** — if two employees tie for 3rd place, `RANK()` returns *both* at rank 3 (you'd get 4 rows for "top 3"), while `ROW_NUMBER()` arbitrarily picks one. Decide up front whether ties should be included or forced to a single winner — this is a business decision, not just a syntax choice.
- **Forgetting `PARTITION BY` entirely** — without it, the ranking runs across the *whole table*, not per group, giving you the global top N instead of the top N per department/category/customer.
- **`LIMIT N` is not the same as "top N per group"** — `LIMIT` caps the entire result set to N rows total; it has no concept of groups. `LIMIT` alone can only give you the global top N, never a per-group top N.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
