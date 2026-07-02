# sql-primer

A practical guide to the SQL patterns that show up again and again in **data analyst** and **data engineer** interviews — what gets asked, the concepts behind it, and free resources to learn SQL from scratch.

**How to use this repo:** For each pattern below, read the folder's `README.md` for the concept, attempt `questions.md` against the [Olist dataset](#practice-dataset), then check your work against `solutions.md`. If you're starting from zero, work through the [free resources](#learn-sql-in-a-week-free-resources) first. See [How to Practice](#how-to-practice) for the full workflow.

## Table of Contents
- [Key Concepts at a Glance](#key-concepts-at-a-glance)
- [Repository Structure](#repository-structure)
- [How to Practice](#how-to-practice)
- [Practice Dataset](#practice-dataset)
- [Learn SQL in a Week (Free Resources)](#learn-sql-in-a-week-free-resources)

## Key Concepts at a Glance

| # | Pattern | In one line | Key SQL |
|---|---------|-------------|---------|
| 1 | [Running Totals](01-running-totals/) | Cumulative sum as you move through ordered rows | `SUM() OVER (ORDER BY ...)` |
| 2 | [Deduplication](02-deduplication/) | Keep one row per key, drop the rest | `ROW_NUMBER() OVER (PARTITION BY ...)` |
| 3 | [Gaps & Islands](03-gaps-and-islands/) | Find streaks or missing sequences over time | `LAG()`, `LEAD()`, grouping by offset |
| 4 | [Join Patterns](04-join-patterns/) | Combine tables, including "what's missing" | `JOIN`, `LEFT JOIN ... WHERE NULL` |
| 5 | [Aggregations](05-aggregations/) | Summarize rows into group-level metrics | `GROUP BY`, `HAVING`, `CASE WHEN` |
| 6 | [Subqueries](06-subqueries/) | Filter using a value computed elsewhere | Correlated / scalar subqueries, `EXISTS` |
| 7 | [CTE](07-cte/) | Name intermediate steps for readability | `WITH ... AS (...)` |
| 8 | [Ranking / Top-N per Group](08-ranking-top-n/) | "Best N per category" queries | `RANK()`, `DENSE_RANK()`, `ROW_NUMBER()` |

Nearly every SQL interview question is one of these 8 patterns wearing a different business story. Recognize the pattern, and the query writes itself.

## Repository Structure

Each pattern has its own folder with three files:

```
sql-primer/
├── DATASET_SETUP.md          ← download & load the Olist dataset locally
├── 01-running-totals/
│   ├── README.md             ← concept, syntax, generic example, gotchas
│   ├── questions.md          ← practice questions against the Olist dataset
│   └── solutions.md          ← worked SQL answers with explanations
├── 02-deduplication/
│   └── ... (same structure)
├── 03-gaps-and-islands/
├── 04-join-patterns/
├── 05-aggregations/
├── 06-subqueries/
├── 07-cte/
└── 08-ranking-top-n/
```

Work through folders in order if you're building up from fundamentals (joins and aggregations before window-function-heavy patterns like running totals and ranking), or jump straight to whichever pattern you're weakest on.

## How to Practice

Reading a solution and understanding it isn't the same as being able to produce it under interview pressure. Get the most out of this repo by treating each pattern as a mini drill:

1. **Read the pattern `README.md` first, SQL editor closed.** Understand the concept, the syntax shape, and the gotchas before you write a single query — most interview mistakes come from missing a gotcha (wrong join type, forgetting to dedupe before aggregating), not from not knowing the syntax exists.
2. **Attempt `questions.md` cold.** Give yourself ~10–15 minutes per question — roughly what you'd get in a live interview. Don't open `solutions.md` yet, even if you get stuck; sitting with the stuck feeling is what builds the instinct.
3. **Run it against the real dataset.** See [Practice Dataset](#practice-dataset) below to get it loaded locally. Sanity-check your output — row count, a few sample values — before assuming it's correct. Real data surfaces edge cases (`NULL`s, duplicates, fan-out joins) that made-up data won't.
4. **Compare against `solutions.md` — read the reasoning, not just the query.** Each solution includes a short note on *why* it's written that way. If your query differs but returns the same result, that's fine; if it differs in approach, understand why the solution chose it (correctness vs. a subtle edge case vs. just style).
5. **Revisit what you got wrong.** After a first pass through all 8 patterns, come back a few days later to the questions you struggled with and redo them without notes. Recognizing the pattern instantly, from a business-question wrapper you haven't seen before, is the actual interview skill.

## Practice Dataset

Every `questions.md` and `solutions.md` in this repo runs against the **[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)** — 100k real orders (2016–2018) across 9 linked tables (orders, order items, payments, reviews, products, customers, sellers, geolocation).

👉 **See [`DATASET_SETUP.md`](DATASET_SETUP.md) for download instructions and how to load it into DuckDB, Postgres, or SQLite.**

| Pattern | How it maps to Olist |
|---|---|
| Running Totals | Cumulative revenue/orders by `order_purchase_timestamp` |
| Deduplication | `order_reviews` has duplicate review rows per order |
| Gaps & Islands | Months/customers with no orders, seller active/inactive streaks |
| Join Patterns | `orders` ↔ `order_items` ↔ `products` ↔ `sellers` ↔ `customers` ↔ `payments` |
| Aggregations | Revenue by category/state, avg delivery time per seller |
| Subqueries | Sellers with above-average review scores |
| CTE | Order → payment → delivery → review funnel |
| Ranking / Top-N | Top 5 sellers by revenue per state, best category per month |

*License: CC BY-NC-SA 4.0 — free for learning/personal projects, not for resale.*

## Learn SQL in a Week (Free Resources)

| Step | Resource | Link |
|---|---|---|
| 1️⃣ Learn the basics | 📺 SQL Course for Beginners (Full Course) | [YouTube](https://www.youtube.com/watch?v=7S_tz1z_5bA) |
| 2️⃣ Practice daily | 💻 Write & run SQL queries online, no setup | [OneCompiler](https://onecompiler.com/mysql) |
| 3️⃣ Analyze real data | 📊 Download datasets and practice real-world analysis | [Kaggle Datasets](https://www.kaggle.com/datasets) |
| 4️⃣ Reference as you go | 📖 SQL syntax reference and tutorials | [W3Schools](https://www.w3schools.com/sql/default.asp) |

**Recommended path:** Watch the course → practice queries daily → download a Kaggle dataset → build a few analysis projects using SQL.
Consistency for 7 days can take you much further than you think. 🚀
