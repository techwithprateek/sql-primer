# Gaps & Islands — Practice Questions

Tables used: `orders`, `order_items`. See [table reference](../DATASET_SETUP.md#table-reference) for columns.

1. **Calendar gaps.** Find every stretch of one or more consecutive calendar days on which **no orders at all** were placed (a full marketplace outage or data gap). Return the start date, end date, and length of each gap.

2. **Seller active-month streaks.** For each seller (`seller_id`), find the longest streak of **consecutive calendar months** in which they sold at least one item. Return `seller_id`, streak start month, streak end month, and streak length in months.

3. **Seller inactivity gaps.** For each seller, find any gap of **60 days or more** between two consecutive sales (i.e. the seller went quiet for 2+ months, then came back). Return the seller, the date their gap started, and how many days it lasted.

4. **Repeat-customer islands.** For customers who placed 3+ orders, find their longest streak of **consecutive months with at least one order**. Return `customer_unique_id`, streak start, streak end, and length — this identifies your most "sticky" repeat buyers.

5. **Order-volume surge islands.** Compute daily order counts, then find every island of **consecutive days where the daily order count is above the dataset's overall daily average** (i.e. sustained demand spikes, not single-day blips).
