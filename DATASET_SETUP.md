# Dataset Setup

All practice questions in this repo run against the **[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)** — 100k real orders (2016–2018) across 9 linked CSV files. This guide gets it running locally so you can execute every query in this repo yourself.

## Option A: DuckDB (fastest, no server, recommended)

[DuckDB](https://duckdb.org/) runs in-process, needs no server or install of a database engine, and can query CSV files directly with full SQL (window functions, CTEs, everything this repo uses).

1. **Download the dataset**
   - Go to [kaggle.com/datasets/olistbr/brazilian-ecommerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
   - Click **Download** (requires a free Kaggle account)
   - Unzip into a folder, e.g. `~/data/olist/`

2. **Install DuckDB**
   ```bash
   # macOS
   brew install duckdb

   # or via pip
   pip install duckdb
   ```

3. **Load the CSVs as tables**

   Start the DuckDB CLI in the folder with your CSVs:
   ```bash
   cd ~/data/olist
   duckdb olist.duckdb
   ```

   Then create a table per CSV:
   ```sql
   CREATE TABLE orders AS SELECT * FROM read_csv_auto('olist_orders_dataset.csv');
   CREATE TABLE order_items AS SELECT * FROM read_csv_auto('olist_order_items_dataset.csv');
   CREATE TABLE order_payments AS SELECT * FROM read_csv_auto('olist_order_payments_dataset.csv');
   CREATE TABLE order_reviews AS SELECT * FROM read_csv_auto('olist_order_reviews_dataset.csv');
   CREATE TABLE customers AS SELECT * FROM read_csv_auto('olist_customers_dataset.csv');
   CREATE TABLE products AS SELECT * FROM read_csv_auto('olist_products_dataset.csv');
   CREATE TABLE sellers AS SELECT * FROM read_csv_auto('olist_sellers_dataset.csv');
   CREATE TABLE geolocation AS SELECT * FROM read_csv_auto('olist_geolocation_dataset.csv');
   CREATE TABLE category_translation AS SELECT * FROM read_csv_auto('product_category_name_translation.csv');
   ```

4. **Verify it worked**
   ```sql
   SHOW TABLES;
   SELECT COUNT(*) FROM orders;   -- should return ~99441
   ```

   Next time, just reopen the same file: `duckdb olist.duckdb` — the tables persist.

## Option B: PostgreSQL

If you'd rather practice on Postgres (closer to what most companies run in production):

1. Download and unzip the dataset as in Option A.
2. Create a database and the 9 tables (`CREATE TABLE ...` matching each CSV's columns — see [Table Reference](#table-reference) below for columns).
3. Load each CSV with `\copy`:
   ```sql
   \copy orders FROM 'olist_orders_dataset.csv' WITH (FORMAT csv, HEADER true);
   ```
   Repeat for each table.

## Option C: Pre-built SQLite file (zero setup)

A ready-to-query SQLite version of this dataset is also available on Kaggle: [E-commerce dataset by Olist (SQLite)](https://www.kaggle.com/datasets/terencicp/e-commerce-dataset-by-olist-as-an-sqlite-database). Download the `.sqlite` file and open it directly with any SQLite client (e.g. `sqlite3 olist.sqlite` or [DB Browser for SQLite](https://sqlitebrowser.org/)) — no CSV loading required.

## Table Reference

| Table | Key columns | Grain |
|---|---|---|
| `orders` | `order_id`, `customer_id`, `order_status`, `order_purchase_timestamp`, `order_delivered_customer_date`, `order_estimated_delivery_date` | One row per order |
| `order_items` | `order_id`, `order_item_id`, `product_id`, `seller_id`, `price`, `freight_value` | One row per item within an order |
| `order_payments` | `order_id`, `payment_type`, `payment_installments`, `payment_value` | One row per payment (an order can have multiple) |
| `order_reviews` | `review_id`, `order_id`, `review_score`, `review_creation_date` | One row per review |
| `customers` | `customer_id`, `customer_unique_id`, `customer_city`, `customer_state` | One row per order-customer (use `customer_unique_id` for the actual person) |
| `products` | `product_id`, `product_category_name`, `product_weight_g` | One row per product |
| `sellers` | `seller_id`, `seller_city`, `seller_state` | One row per seller |
| `geolocation` | `geolocation_zip_code_prefix`, `geolocation_lat`, `geolocation_lng` | One row per zip code (many rows per prefix) |
| `category_translation` | `product_category_name`, `product_category_name_english` | Lookup table |

> Note: `customer_id` in `orders` is unique **per order**, not per person. To identify a returning customer, join to `customers` and use `customer_unique_id`.

## Next Steps

Once your tables are loaded, head to any pattern folder (e.g. [`01-running-totals/`](01-running-totals/)) — each has a `README.md` explaining the concept, a `questions.md` with Olist-based practice questions, and a `solutions.md` with worked answers.
