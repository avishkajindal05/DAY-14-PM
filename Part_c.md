````markdown

# Q1 — Difference Between `RANK()` and `DENSE_RANK()`

## Concept

Both functions assign **ranking positions** within a partition of rows based on an ordering condition.

### `RANK()`

- Assigns the same rank to tied rows.
- **Skips rank numbers** after ties.

### `DENSE_RANK()`

- Assigns the same rank to tied rows.
- **Does NOT skip numbers**.

---

## Example Table

| employee | salary |
|---|---|
| Alice | 90000 |
| Bob | 85000 |
| Charlie | 85000 |
| David | 80000 |

---

## SQL Query

```sql
SELECT 
    employee,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank_value,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank_value
FROM employees;
````

---

## Output

| employee | salary | rank_value | dense_rank_value |
| -------- | ------ | ---------- | ---------------- |
| Alice    | 90000  | 1          | 1                |
| Bob      | 85000  | 2          | 2                |
| Charlie  | 85000  | 2          | 2                |
| David    | 80000  | 4          | 3                |

Notice:

* `RANK()` skips **3**
* `DENSE_RANK()` keeps sequential numbering

---

## Business Context Where This Matters

### Example: Sales Leaderboard

Suppose a company awards bonuses to **top 3 salespeople**.

If using:

### `RANK()`

```
Salesperson A → Rank 1
Salesperson B → Rank 2
Salesperson C → Rank 2
Salesperson D → Rank 4
```

Only **three employees qualify (1,2,2)**.

### `DENSE_RANK()`

```
A → 1
B → 2
C → 2
D → 3
```

Now **four employees qualify (1,2,2,3)**.

Thus the choice affects:

* **bonus distribution**
* **leaderboards**
* **top-N analysis**

---

# Q2 — Users With Purchases in 3 Consecutive Months

Table:

```
transactions(user_id, transaction_date, amount)
```

Goal: Find users with **purchases in 3 or more consecutive months**.

---

## SQL Query

```sql
WITH monthly_activity AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', transaction_date) AS month
    FROM transactions
    GROUP BY user_id, DATE_TRUNC('month', transaction_date)
),

ordered_months AS (
    SELECT
        user_id,
        month,
        LAG(month) OVER (
            PARTITION BY user_id 
            ORDER BY month
        ) AS prev_month
    FROM monthly_activity
),

month_diff AS (
    SELECT
        user_id,
        month,
        CASE
            WHEN month - prev_month = INTERVAL '1 month'
            THEN 1
            ELSE 0
        END AS consecutive_flag
    FROM ordered_months
)

SELECT user_id
FROM month_diff
GROUP BY user_id
HAVING SUM(consecutive_flag) >= 2;
```

---

## Explanation

Steps:

1. **monthly_activity**

   Extract distinct months for each user.

2. **ordered_months**

   Use `LAG()` to compare current month with previous month.

3. **month_diff**

   Identify consecutive months.

4. Final step:

   If a user has **2 consecutive gaps**, they have **3 consecutive months**.

---

## Example Transactions

| user_id | transaction_date | amount |
| ------- | ---------------- | ------ |
| 1       | 2024-01-10       | 100    |
| 1       | 2024-02-05       | 80     |
| 1       | 2024-03-12       | 50     |
| 2       | 2024-01-15       | 60     |
| 2       | 2024-03-10       | 40     |
| 3       | 2024-02-10       | 90     |
| 3       | 2024-03-15       | 70     |
| 3       | 2024-04-18       | 110    |

---

## Output

| user_id |
| ------- |
| 1       |
| 3       |

These users made purchases in **three consecutive months**.

---

# Q3 — Optimize Correlated Subquery Using Window Functions

## Original Query (Correlated Subquery)

```sql
SELECT name, salary
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department = e1.department
);
```

Problem:

* The subquery executes **once per row**, which can be inefficient for large tables.

---

# Optimized Query Using Window Function

```sql
SELECT name, salary
FROM (
    SELECT 
        name,
        salary,
        AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
    FROM employees
) t
WHERE salary > dept_avg_salary;
```

---

# Explanation

The window function:

```
AVG(salary) OVER (PARTITION BY department)
```

calculates the **department average once per partition**, avoiding repeated subqueries.

Benefits:

* Fewer scans
* Better execution plans
* Often converts to **single-pass aggregation**

---

# Example Table

| name    | department   | salary |
| ------- | ------------ | ------ |
| Alice   | Engineering  | 90000  |
| Bob     | Engineering  | 75000  |
| Charlie | Data Science | 82000  |
| David   | Data Science | 67000  |

---

## Output

| name    | salary |
| ------- | ------ |
| Alice   | 90000  |
| Charlie | 82000  |

These employees earn **above their department's average salary**.


```
```
