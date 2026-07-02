# 3. Gaps & Islands

## Concept

"Gaps and islands" is the classic name for a family of problems about **ordered, potentially-interrupted sequences**: find the runs of consecutive values (*islands*) and/or the missing values between them (*gaps*). Think: consecutive login days, active subscription streaks, sensor readings that drop out, or months with no sales.

The core trick: an "island" is a group of rows where the value increases in lockstep with its row position. If you subtract a running row number from the ordered value, every row in the same island produces the **same constant** — because both are advancing by the same amount each step. That constant becomes a ready-made `GROUP BY` key.

## Syntax

```sql
WITH numbered AS (
    SELECT
        *,
        <ordering_value> - ROW_NUMBER() OVER (
            PARTITION BY <group_column>
            ORDER BY <ordering_value>
        ) AS island_id
    FROM <table>
)
SELECT
    <group_column>,
    MIN(<ordering_value>) AS island_start,
    MAX(<ordering_value>) AS island_end,
    COUNT(*)              AS island_length
FROM numbered
GROUP BY <group_column>, island_id;
```

The alternative approach uses `LAG()`/`LEAD()` directly: compare each row to the previous one, and flag a new island whenever the gap between them exceeds one step.

## Generic Example

```sql
WITH streaks AS (
    SELECT
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ) * INTERVAL '1 day') AS grp
    FROM logins
)
SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*)         AS streak_length
FROM streaks
GROUP BY user_id, grp;
```

Every row in the same unbroken streak of consecutive days lands on the same `grp` value, because `login_date` and the row number both increase by exactly one day per row within the streak.

## Common Gotchas

- **This only works on genuinely sequential data** — dates, integer ids, or anything with a fixed step size. It doesn't work directly on arbitrary timestamps; truncate to the grain you care about first (`DATE(...)`, `DATE_TRUNC('month', ...)`).
- **Finding gaps instead of islands** — once you have islands, the gaps are just the space *between* one island's end and the next island's start. Use `LEAD(island_start) OVER (ORDER BY island_start)` on the collapsed island table.
- **Partitioning matters** — forgetting `PARTITION BY user_id` (or seller/customer/etc.) merges everyone's sequence together into one meaningless mega-island.
- **Off-by-one in streak length** — `MAX - MIN` gives you a date *range*, not a count; use `COUNT(*)` for the actual number of rows in the island, since a streak with a data quality issue could have fewer rows than its date span implies.

## Practice

- [`questions.md`](questions.md) — Olist-based questions for this pattern
- [`solutions.md`](solutions.md) — worked SQL solutions

See [`DATASET_SETUP.md`](../DATASET_SETUP.md) to get the dataset running locally first.
