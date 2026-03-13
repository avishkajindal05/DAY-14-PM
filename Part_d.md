```markdown
# Senior Data Engineer SQL Interview Questions (Window Functions & CTEs)

These questions reflect **real senior-level SQL interview expectations**.  
They test understanding of **window functions, CTE design, query planning, and edge cases**.

---

# Q1 — Identify Employees With the Largest Salary Increase

### Problem

Given a table:

```

employee_salaries(emp_id, salary, effective_date)

````

Each row represents a salary change.

Write a query that returns **the largest salary increase for each employee**.

---

## Expected SQL Answer

```sql
WITH salary_changes AS (
    SELECT
        emp_id,
        salary,
        effective_date,
        salary - LAG(salary) OVER (
            PARTITION BY emp_id
            ORDER BY effective_date
        ) AS salary_change
    FROM employee_salaries
)

SELECT
    emp_id,
    MAX(salary_change) AS largest_increase
FROM salary_changes
GROUP BY emp_id;
````

---

## Explanation

1. `LAG()` retrieves the previous salary.
2. The difference between current and previous salary gives the **salary change**.
3. We then **aggregate using MAX()** to find the largest increase.

---

## Common Candidate Mistake

Using `MAX(salary) - MIN(salary)`.

Example incorrect solution:

```sql
SELECT emp_id, MAX(salary) - MIN(salary)
FROM employee_salaries
GROUP BY emp_id;
```

Why this is wrong:

* It ignores **salary order over time**
* It calculates **total change**, not **largest single increase**

---

# Q2 — Detect Streaks of Consecutive Active Days

### Problem

Given a table:

```
user_activity(user_id, activity_date)
```

Return users who were active for **5 or more consecutive days**.

---

## Expected SQL Answer

```sql
WITH ordered_activity AS (
    SELECT
        user_id,
        activity_date,
        activity_date - 
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY activity_date
        ) * INTERVAL '1 day' AS grp
    FROM user_activity
)

SELECT
    user_id,
    COUNT(*) AS streak_length
FROM ordered_activity
GROUP BY user_id, grp
HAVING COUNT(*) >= 5;
```

---

## Explanation

The trick:

```
date - row_number()
```

creates a **constant group key for consecutive dates**.

Example:

| date | row_number | date-row_number |
| ---- | ---------- | --------------- |
| Jan1 | 1          | Dec31           |
| Jan2 | 2          | Dec31           |
| Jan3 | 3          | Dec31           |

These rows belong to the **same streak group**.

---

## Common Candidate Mistake

Trying to solve using only `LAG()`:

```sql
WHERE activity_date - LAG(activity_date) = 1
```

Why this fails:

* It detects **pairs**, not **streak length**
* Cannot count longer sequences properly

---

# Q3 — Find Top 2 Salaries Per Department (Efficiently)

### Problem

Given:

```
employees(emp_id, name, department, salary)
```

Return the **top 2 highest-paid employees per department**.

---

## Expected SQL Answer

```sql
SELECT *
FROM (
    SELECT
        emp_id,
        name,
        department,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees
) ranked
WHERE salary_rank <= 2;
```

---

## Explanation

* `PARTITION BY department` ranks employees **within each department**
* `ORDER BY salary DESC` ranks highest salaries first
* `DENSE_RANK()` handles ties properly

---

## Common Candidate Mistake

Using `LIMIT 2`.

Incorrect:

```sql
SELECT *
FROM employees
ORDER BY salary DESC
LIMIT 2;
```

Why wrong:

* Returns **top 2 overall**, not **top 2 per department**

Another common mistake:

Using `ROW_NUMBER()` when ties should be included.

Example problem:

| dept | salary |
| ---- | ------ |
| Eng  | 100    |
| Eng  | 100    |
| Eng  | 90     |

`ROW_NUMBER()` might exclude tied values unfairly.

---

# Key Skills These Questions Test

Senior data engineer SQL interviews often evaluate:

* **Advanced window functions**
* **Temporal reasoning**
* **Sequence detection**
* **Query optimization**
* **Correct partitioning logic**

These patterns frequently appear in **analytics pipelines, feature engineering, and data warehouse queries**.

---

# MISTAKES MADE BY ME

OPTIMIZATION OF QUERIES IS MISSING, COMPARED TO THE QUERIES GIVE BY CHATGPT.


```
```
