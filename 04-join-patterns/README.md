# 4. Join Patterns

## Concept

A join combines rows from two tables based on a matching condition. The type of join you choose encodes a business question about **what to do when a match is missing** — that's the part interviewers actually care about, not the join syntax itself.

| Join | Keeps | Use when |
|---|---|---|
| `INNER JOIN` | Only rows with a match on both sides | You only want records that exist in both tables |
| `LEFT JOIN` | All rows from the left table, matched or not | You want everything from table A, enriched with B where available |
| `RIGHT JOIN` | All rows from the right table, matched or not | Same as `LEFT JOIN` with the tables reversed (rarely used — most people just flip to `LEFT JOIN`) |
| `FULL JOIN` | All rows from both tables, matched or not | You need to see mismatches on either side |
| Anti-join | Rows from A with **no** match in B | "Find X that never happened Y" — e.g. customers with no orders |
| Self-join | A table joined to itself | Comparing rows within the same table — e.g. employees to their manager (also in the employees table) |

## Syntax

```sql
SELECT ...
FROM a
LEFT JOIN b ON a.key = b.key;
```

**Anti-join** (the pattern most people forget): a `LEFT JOIN` where you then filter for the *unmatched* rows:

```sql
SELECT a.*
FROM a
LEFT JOIN b ON a.key = b.key
WHERE b.key IS NULL;
```

## Generic Example

```sql
-- Customers who have never placed an order
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

The `LEFT JOIN` keeps every customer regardless of match; for customers with no orders, every column from `orders` comes back `NULL`. Filtering on `o.order_id IS NULL` isolates exactly those "never matched" rows.

## Common Gotchas

- **Filtering a `LEFT JOIN` in the `WHERE` clause accidentally turns it into an `INNER JOIN`** — e.g. `WHERE o.status = 'delivered'` on a `LEFT JOIN` silently drops all the unmatched rows you joined to keep. If you need to filter the *right* table's rows while still keeping unmatched left rows, move the condition into the `ON` clause instead.
- **Fan-out / row duplication** — joining a "one" table to a "many" table (e.g. `orders` to `order_items`) multiplies each order row by its item count. If you then `SUM()` a column from the "one" side (like a payment total), you'll overcount. Aggregate one side down to the right grain *before* joining, or aggregate after with `DISTINCT`/pre-grouped CTEs.
- **`NULL` doesn't equal `NULL`** — a join condition `a.key = b.key` will never match two `NULL` keys. If your join key can be `NULL`, that's a silent row-dropper.
- **Self-joins need aliases** — `FROM employees a JOIN employees b ON a.manager_id = b.employee_id` — without distinct aliases, SQL can't tell which "copy" of the table a column belongs to.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
