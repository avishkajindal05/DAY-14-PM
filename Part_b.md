
# Recursive CTE in SQL — Generating Series and Filling Missing Dates
Goals:
1. Generate a number series **1–100** using a recursive CTE (without hardcoding a list).
2. Use a recursive CTE to generate **continuous dates**.
3. Fill **missing dates in a sparse time series** so that dates with no orders show `revenue = 0`.

---

# Part 1 — Generate Numbers 1 to 100 Using Recursive CTE

## SQL Query

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1
    FROM numbers
    WHERE n < 100
)
SELECT n
FROM numbers;
```

## How it Works

1. **Base case**
   ```sql
   SELECT 1
   ```
   Starts the sequence.

2. **Recursive step**
   ```sql
   SELECT n + 1 FROM numbers WHERE n < 100
   ```

3. The query repeatedly adds 1 until **100 is reached**.

---

## Output (first 10 rows)

| n |
|---|
|1|
|2|
|3|
|4|
|5|
|6|
|7|
|8|
|9|
|10|
...
|100|

---

# Part 2 — Sparse Time Series Example

Suppose we have a table:

```sql
orders
```

| order_date | revenue |
|---|---|
|2024-01-01|500|
|2024-01-03|200|
|2024-01-06|900|

Notice **missing dates** (Jan 2, Jan 4, Jan 5).

Goal: return a **continuous time series**.

---

# Step 1 — Generate Date Series Using Recursive CTE

```sql
WITH RECURSIVE date_series AS (
    SELECT DATE '2024-01-01' AS dt
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < DATE '2024-01-07'
)
SELECT dt
FROM date_series;
```

---

# Step 2 — Fill Missing Dates

```sql
WITH RECURSIVE date_series AS (
    SELECT DATE '2024-01-01' AS dt
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < DATE '2024-01-07'
)

SELECT 
    d.dt AS order_date,
    COALESCE(o.revenue,0) AS revenue
FROM date_series d
LEFT JOIN orders o
ON d.dt = o.order_date
ORDER BY d.dt;
```

---

# Final Output

| order_date | revenue |
|---|---|
|2024-01-01|500|
|2024-01-02|0|
|2024-01-03|200|
|2024-01-04|0|
|2024-01-05|0|
|2024-01-06|900|
|2024-01-07|0|

---

# Why This Is Important for Data Engineering

This pattern is commonly used for:

- **Time series completion**
- **Data warehouse gap filling**
- **Generating surrogate sequences**
- **Calendar tables**
- **ETL pipelines**

In PostgreSQL production systems, this approach is often replaced with:

```sql
generate_series()
```

Example:

```sql
SELECT *
FROM generate_series('2024-01-01'::date, '2024-01-07', '1 day');
```

However, **recursive CTEs are database‑portable**, which is why interviewers often ask this problem.

---

# Key Interview Takeaways

- Recursive CTEs have **two parts**:
  - Base case
  - Recursive step
- Used for **hierarchies, graph traversal, and sequences**
- Powerful for **time‑series gap filling**
- Frequently asked in **Data Engineer interviews**

