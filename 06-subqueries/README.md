# 6. Subqueries

## Concept

A subquery is a query nested inside another query, used to compute a value or a set of values that the outer query then filters or selects against. The key distinction to internalize:

- **Non-correlated subquery** — runs once, independently of the outer query, and produces a fixed value or list (e.g. "the overall average salary"). The outer query then compares every row to that one fixed result.
- **Correlated subquery** — references a column from the outer query, so it conceptually re-runs *once per outer row* (e.g. "the average salary in *this row's* department"). This is what makes "compare each row to its own group's average" possible without a full `GROUP BY`/join round-trip.

Subqueries can appear in three places: the `WHERE` clause (filtering), the `SELECT` list (a scalar subquery producing one value per row), or the `FROM` clause (an inline, unnamed table — though a named CTE is almost always more readable for this).

## Syntax

**Correlated subquery** (references the outer table):
```sql
SELECT *
FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees WHERE department_id = e.department_id
);
```

**`EXISTS`** — checks only for presence of a matching row, without pulling back any values; often faster than `IN` because it can stop at the first match:
```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

**`IN`** — compares against a list of values from a non-correlated subquery:
```sql
SELECT * FROM products WHERE product_id IN (SELECT product_id FROM order_items);
```

## Generic Example

```sql
SELECT employee_id, name, salary, department_id
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e.department_id
);
```

The inner query's `WHERE department_id = e.department_id` is what makes this correlated — `e` refers to the outer row currently being evaluated, so the average is recomputed per department rather than once for the whole table.

## Common Gotchas

- **`IN` with `NULL`s** — if the subquery's result list contains even one `NULL`, `NOT IN` can silently return zero rows for the entire query (a classic, hard-to-spot bug). Prefer `NOT EXISTS` over `NOT IN` when the subquery column can contain `NULL`s.
- **Correlated subqueries can be slow** — because they conceptually re-run per outer row, they don't always scale well on large tables. A window function (`AVG(salary) OVER (PARTITION BY department_id)`) often expresses the same "compare to my group" logic in a single pass and is usually faster — see [CTE](../07-cte/) and revisit [Ranking / Top-N](../08-ranking-top-n/) for when window functions replace subqueries entirely.
- **Scalar subquery must return exactly one row** — if it can return more than one, the database will error at runtime. Use `LIMIT 1` with a deterministic `ORDER BY`, or an aggregate function, to guarantee a single value.
- **Readability at scale** — deeply nested subqueries get hard to read fast. A CTE with a descriptive name usually communicates the same logic more clearly.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
