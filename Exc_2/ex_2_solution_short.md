# Exercise 2 — Solutions

## Activity 2.1 — E-commerce

### Create table
```sql
CREATE TABLE orders (
    customer_name TEXT,
    product_category TEXT,
    quantity INTEGER,
    price_per_unit NUMERIC(10,2),
    order_date DATE,
    country TEXT
);
```

### A. Highest unit price
```sql
SELECT * FROM orders ORDER BY price_per_unit DESC LIMIT 1;
```
**Result:** Emma Brown | Automotive | 3 | 2000.00 | 2024-10-11 | Italy

### B. Top 3 categories by quantity
```sql
SELECT product_category, SUM(quantity) AS total_quantity
FROM orders GROUP BY product_category
ORDER BY total_quantity DESC LIMIT 3;
```
**Result:** Health & Beauty: 300842; Electronics: 300804; Toys: 300598.

### C. Revenue per category
```sql
SELECT product_category, SUM(price_per_unit * quantity) AS total_revenue
FROM orders GROUP BY product_category ORDER BY total_revenue DESC;
```

| Category | Revenue |
|---|---:|
| Automotive | 306589798.86 |
| Electronics | 241525009.45 |
| Home & Garden | 78023780.09 |
| Sports | 61848990.83 |
| Health & Beauty | 46599817.89 |
| Office Supplies | 38276661.64 |
| Fashion | 31566368.22 |
| Toys | 23271039.02 |
| Grocery | 15268355.66 |
| Books | 12731976.04 |

### D. Top 5 customers by spending
```sql
SELECT customer_name, SUM(price_per_unit * quantity) AS total_spending,
       COUNT(*) AS number_of_orders
FROM orders GROUP BY customer_name
ORDER BY total_spending DESC LIMIT 5;
```

| Customer | Spending | Orders |
|---|---:|---:|
| Carol Taylor | 991179.18 | 1028 |
| Nina Lopez | 975444.95 | 980 |
| Daniel Jackson | 959344.48 | 1033 |
| Carol Lewis | 947708.57 | 943 |
| Daniel Young | 946030.14 | 973 |

### E. Explanation
The generator randomly combines 32 first names and 33 last names (1056 combinations). Across 1 million orders, names repeat about 947 times on average. Grouping by name therefore produces around 1000 orders and unusually high spending per name.

## Activity 2.2 — Slow self-join

### Step 1. Measurements and prediction
```sql
CREATE TABLE people_50k AS SELECT * FROM people_big WHERE id <= 50000;
CREATE TABLE people_100k AS SELECT * FROM people_big WHERE id <= 100000;
CREATE TABLE people_200k AS SELECT * FROM people_big WHERE id <= 200000;

-- Run with each smaller table name:
SELECT COUNT(*) FROM people_50k p1
JOIN people_50k p2 ON p1.country = p2.country;
```

| Rows | Matching pairs | Time |
|---|---:|---:|
| 50000 | 27501822 | 1219.960 ms |
| 100000 | 109946508 | 5197.997 ms |
| 200000 | 439395066 | 32131.117 ms |

Doubling the input gives about **4× more pairs**; time increased **4.26×**, then **6.18×**. For 1M rows, predict **~10.98 billion pairs** and **~13.4 minutes** using quadratic scaling from 200k (estimate only; full join not run).

### Step 2. Index test
```sql
CREATE INDEX idx_people_100k_country ON people_100k(country);
ANALYZE people_100k;
EXPLAIN ANALYZE
SELECT COUNT(*) FROM people_100k p1
JOIN people_100k p2 ON p1.country = p2.country;
```
**Before:** 5197.997 ms. **After:** 8602.431 ms. The plan used parallel index-only scans and a parallel hash join. The index did not help in this run: the expensive part is processing ~110 million matching pairs.

### Step 3. Rewrite without a join
For each country with **k** people, the self-join counts **k²** pairs.
```sql
SELECT SUM(cnt * cnt) AS total_pairs
FROM (SELECT country, COUNT(*) AS cnt
      FROM people_100k GROUP BY country) sub;
```
**100k:** 109946508 pairs in **6.055 ms** (same count; ~858× faster than original).

```sql
SELECT SUM(cnt * cnt) AS total_pairs
FROM (SELECT country, COUNT(*) AS cnt
      FROM people_big GROUP BY country) sub;
```
**1M:** 10983941260 pairs in **250.634 ms**, versus ~13.4 minutes predicted for the full self-join.

### Step 4. Discussion
- **Rewrite first:** Aggregating by country avoids generating all matching pairs. An index or more hardware cannot remove that work.
- **If actual pairs are needed:** Filter first, process in batches, and consider parallel/distributed execution. A bigger machine or cluster may help, but billions of output rows remain expensive to compute and transfer.
- **OLTP limits:** Large analytical joins compete with transactions for CPU, memory and I/O. In cloud systems, use a separate analytical replica or data warehouse for heavy workloads.
