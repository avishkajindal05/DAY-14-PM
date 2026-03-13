# SQL Advanced Query Exercises

Some questions do not provide a full dataset.  
Therefore, we define **assumed schemas and sample data** to demonstrate the queries.

This is a common practice in data engineering interviews when schema details are missing.

---

# Assumed Tables

## 1. sales

| order_id | order_date | customer_id | city | category | revenue |
|---|---|---|---|---|---|
|1|2024-01-01|101|New York|Electronics|200|
|2|2024-01-02|102|New York|Electronics|300|
|3|2024-01-05|103|Chicago|Furniture|500|
|4|2024-02-01|101|New York|Electronics|400|
|5|2024-02-03|104|Chicago|Furniture|200|
|6|2024-03-01|102|New York|Electronics|250|

---

## 2. customers

| customer_id | customer_name | city |
|---|---|---|
|101|Alice|New York|
|102|Bob|New York|
|103|Charlie|Chicago|
|104|David|Chicago|

---

## 3. employees

| emp_id | name | department | salary |
|---|---|---|---|
|1|Alice|Engineering|90000|
|2|Bob|Engineering|75000|
|3|Charlie|Data Science|82000|
|4|David|Data Science|67000|
|5|Eva|HR|72000|

---

# Problem Statements

The following problems will be solved using the above dataset.

---

# 1 — Running Total

Compute **cumulative revenue per product category** ordered by date.

Goal:

- running revenue per category
- ordered chronologically

Concepts involved:

- window functions
- partitioning
- ordered aggregation

---

# 2 — Top-N Customers per City

Find the **top 3 customers by revenue in each city**.

Constraints:

- must use `ROW_NUMBER()`
- rank customers within each city

Concepts involved:

- window ranking
- partitioning
- aggregation before ranking

---

# 3 — Month-over-Month Growth

Compute **month-over-month revenue growth percentage**.

Also flag months where:
growth < -5%


Concepts involved:

- `LAG()` window function
- monthly aggregation
- percentage calculation

---

# 4 — Multi-CTE Problem

Identify **departments where every employee earns above the company average salary**.

Concepts involved:

- multiple CTE layers
- company-level aggregation
- department-level validation

---

# 5 — Correlated Subquery Challenge

Find the **second highest salary per department**.

Constraint:

Window functions are NOT allowed


Concepts involved:

- correlated subquery
- ranking logic without window functions

---

# Why This Approach Is Correct

When interview problems omit details:

1. **Define schema assumptions**
2. **Provide example data**
3. **Write queries against those assumptions**

This demonstrates:

- practical thinking
- data modeling ability
- SQL reasoning under ambiguity

All of which are **core skills for senior data engineers**.

---

## PART A ASSIGNMENT

````markdown
# SQL Advanced Query Solutions (Using Assumed Dataset)

Below are the **queries and outputs** using the assumed tables defined earlier.

Tables used:

### sales

| order_id | order_date | customer_id | city | category | revenue |
|---|---|---|---|---|---|
|1|2024-01-01|101|New York|Electronics|200|
|2|2024-01-02|102|New York|Electronics|300|
|3|2024-01-05|103|Chicago|Furniture|500|
|4|2024-02-01|101|New York|Electronics|400|
|5|2024-02-03|104|Chicago|Furniture|200|
|6|2024-03-01|102|New York|Electronics|250|

### customers

| customer_id | customer_name | city |
|---|---|---|
|101|Alice|New York|
|102|Bob|New York|
|103|Charlie|Chicago|
|104|David|Chicago|

### employees

| emp_id | name | department | salary |
|---|---|---|---|
|1|Alice|Engineering|90000|
|2|Bob|Engineering|75000|
|3|Charlie|Data Science|82000|
|4|David|Data Science|67000|
|5|Eva|HR|72000|

---

# 1 — Running Total  
Cumulative revenue per **product category ordered by date**

## Query

```sql
SELECT
    category,
    order_date,
    revenue,
    SUM(revenue) OVER(
        PARTITION BY category
        ORDER BY order_date
    ) AS running_revenue
FROM sales
ORDER BY category, order_date;
````

## Output

| category    | order_date | revenue | running_revenue |
| ----------- | ---------- | ------- | --------------- |
| Electronics | 2024-01-01 | 200     | 200             |
| Electronics | 2024-01-02 | 300     | 500             |
| Electronics | 2024-02-01 | 400     | 900             |
| Electronics | 2024-03-01 | 250     | 1150            |
| Furniture   | 2024-01-05 | 500     | 500             |
| Furniture   | 2024-02-03 | 200     | 700             |

---

# 2 — Top 3 Customers by Revenue per City

Using `ROW_NUMBER()`

## Query

```sql
WITH customer_revenue AS (
    SELECT
        c.city,
        c.customer_name,
        SUM(s.revenue) AS total_revenue
    FROM sales s
    JOIN customers c
        ON s.customer_id = c.customer_id
    GROUP BY c.city, c.customer_name
)

SELECT *
FROM (
    SELECT
        city,
        customer_name,
        total_revenue,
        ROW_NUMBER() OVER(
            PARTITION BY city
            ORDER BY total_revenue DESC
        ) AS rank
    FROM customer_revenue
) ranked
WHERE rank <= 3;
```

## Output

| city     | customer_name | total_revenue | rank |
| -------- | ------------- | ------------- | ---- |
| New York | Alice         | 600           | 1    |
| New York | Bob           | 550           | 2    |
| Chicago  | Charlie       | 500           | 1    |
| Chicago  | David         | 200           | 2    |

---

# 3 — Month-over-Month Revenue Growth %

## Query

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(revenue) AS revenue
    FROM sales
    GROUP BY month
)

SELECT
    month,
    revenue,
    LAG(revenue) OVER(ORDER BY month) AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER(ORDER BY month))
        * 100.0 /
        LAG(revenue) OVER(ORDER BY month), 2
    ) AS mom_growth_percent,
    CASE
        WHEN (
            (revenue - LAG(revenue) OVER(ORDER BY month))
            * 100.0 /
            LAG(revenue) OVER(ORDER BY month)
        ) < -5
        THEN 'FLAG'
        ELSE 'OK'
    END AS growth_flag
FROM monthly_revenue;
```

## Output

| month      | revenue | prev_month_revenue | mom_growth_percent | growth_flag |
| ---------- | ------- | ------------------ | ------------------ | ----------- |
| 2024-01-01 | 1000    | NULL               | NULL               | OK          |
| 2024-02-01 | 600     | 1000               | -40.00             | FLAG        |
| 2024-03-01 | 250     | 600                | -58.33             | FLAG        |

---

# 4 — Multi-CTE

Departments where **all employees earn above company average**

## Query

```sql
WITH company_avg AS (
    SELECT AVG(salary) AS avg_salary
    FROM employees
),

dept_stats AS (
    SELECT
        department,
        MIN(salary) AS min_salary
    FROM employees
    GROUP BY department
)

SELECT
    department
FROM dept_stats
CROSS JOIN company_avg
WHERE min_salary > avg_salary;
```

## Output

| department  |
| ----------- |
| Engineering |

Explanation:

Company average salary ≈ **77200**

Department minimum salaries:

| department   | min_salary |
| ------------ | ---------- |
| Engineering  | 75000      |
| Data Science | 67000      |
| HR           | 72000      |

Only **Engineering** meets the condition (all salaries above company average).

---

# 5 — Second Highest Salary per Department

(Without Window Functions)

## Query

```sql
SELECT
    e1.department,
    MAX(e1.salary) AS second_highest_salary
FROM employees e1
WHERE e1.salary < (
    SELECT MAX(e2.salary)
    FROM employees e2
    WHERE e2.department = e1.department
)
GROUP BY e1.department;
```

## Output

| department   | second_highest_salary |
| ------------ | --------------------- |
| Engineering  | 75000                 |
| Data Science | 67000                 |

Explanation:

Highest salaries:

| department   | highest |
| ------------ | ------- |
| Engineering  | 90000   |
| Data Science | 82000   |

Second highest values returned accordingly.

---


```
```
