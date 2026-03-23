# SQL Deep Dive

SQL is the most valuable query language a software engineer can know. This document covers the features that separate a junior developer writing basic `SELECT` statements from a senior developer who solves complex business problems in a single query.

---

## Joins

Joins combine rows from two or more tables based on a related column. Understanding joins is the foundation of relational querying.

### INNER JOIN

Returns only rows that have a match in both tables.

```sql
-- Get all orders with user names (only users who have orders)
SELECT o.id AS order_id, u.name, o.total, o.created_at
FROM orders o
INNER JOIN users u ON o.user_id = u.id;
```

### LEFT JOIN

Returns all rows from the left table, with NULLs for the right table when there is no match.

```sql
-- Get all users with their order count (including users with 0 orders)
SELECT u.name, u.email, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.name, u.email;
```

**Pitfall:** Using `COUNT(*)` instead of `COUNT(o.id)` with LEFT JOIN. `COUNT(*)` counts NULL rows, so users with no orders would show `order_count = 1`. `COUNT(o.id)` correctly returns 0 because `o.id` is NULL.

### RIGHT JOIN and FULL OUTER JOIN

Right join is a left join with the tables swapped. Full outer join returns all rows from both tables. In practice, left join covers 95% of use cases. If you find yourself writing a right join, swap the table order and use a left join instead.

### CROSS JOIN

Produces the cartesian product of two tables. Rarely needed, but useful for generating combinations.

```sql
-- Generate a report template: every user x every month
SELECT u.name, m.month
FROM users u
CROSS JOIN generate_series(
    '2024-01-01'::date,
    '2024-12-01'::date,
    '1 month'::interval
) AS m(month);
```

### Self joins

A table joined to itself. Common for hierarchical data.

```sql
-- Find employees and their managers
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

## Window Functions

Window functions perform a calculation across a set of rows related to the current row, without collapsing the result set. They are among the most powerful features in SQL.

### Syntax

```sql
function_name() OVER (
    [PARTITION BY column_list]
    [ORDER BY column_list]
    [frame_clause]
)
```

- **PARTITION BY** divides rows into groups (like GROUP BY, but without collapsing)
- **ORDER BY** defines the order within each partition
- **Frame clause** defines which rows relative to the current row are included

### ROW_NUMBER, RANK, DENSE_RANK

```sql
-- Rank users by total spending within each region
SELECT
    region,
    user_id,
    total_spent,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_spent DESC) AS row_num,
    RANK()       OVER (PARTITION BY region ORDER BY total_spent DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY region ORDER BY total_spent DESC) AS dense_rank
FROM user_stats;
```

| Difference | Behavior on ties |
|-----------|------------------|
| **ROW_NUMBER** | Assigns unique numbers; ties get arbitrary order |
| **RANK** | Ties get the same rank; next rank is skipped (1, 1, 3) |
| **DENSE_RANK** | Ties get the same rank; next rank is not skipped (1, 1, 2) |

### The "latest per group" pattern

One of the most common real-world uses: get the most recent record per group.

```sql
-- Each user's most recent order
SELECT * FROM (
    SELECT
        o.*,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders o
) sub
WHERE rn = 1;
```

**Real-world use case:** An admin dashboard showing each customer's last login, last purchase, or last support ticket.

### Running totals and moving averages

```sql
-- Cumulative daily revenue
SELECT
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date) AS cumulative_revenue
FROM daily_revenue;

-- 7-day moving average of daily revenue
SELECT
    date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_revenue;
```

### LAG and LEAD

Access the previous or next row's value without a self-join.

```sql
-- Month-over-month revenue growth
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    revenue - LAG(revenue) OVER (ORDER BY month) AS growth,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100,
        1
    ) AS growth_percent
FROM monthly_revenue;
```

### NTILE: percentile buckets

```sql
-- Divide users into spending quartiles
SELECT
    user_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent) AS spending_quartile
FROM user_stats;
```

---

## Common Table Expressions (CTEs)

CTEs let you name intermediate query results, making complex queries readable. They are defined with the `WITH` keyword.

### Basic CTE

```sql
WITH active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active' AND last_login > NOW() - INTERVAL '30 days'
)
SELECT au.name, COUNT(o.id) AS recent_orders
FROM active_users au
LEFT JOIN orders o ON au.id = o.user_id AND o.created_at > NOW() - INTERVAL '30 days'
GROUP BY au.name
ORDER BY recent_orders DESC;
```

### Chained CTEs for complex reporting

```sql
-- Monthly revenue report with growth rates and running totals
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total) AS revenue,
        COUNT(*) AS order_count,
        COUNT(DISTINCT user_id) AS unique_customers
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', created_at)
),
with_growth AS (
    SELECT
        month,
        revenue,
        order_count,
        unique_customers,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
        revenue - LAG(revenue) OVER (ORDER BY month) AS revenue_growth
    FROM monthly_revenue
),
with_running_total AS (
    SELECT
        *,
        SUM(revenue) OVER (ORDER BY month) AS ytd_revenue,
        ROUND(
            revenue_growth / NULLIF(prev_revenue, 0) * 100, 1
        ) AS growth_percent
    FROM with_growth
)
SELECT * FROM with_running_total ORDER BY month;
```

### Recursive CTEs

Recursive CTEs query hierarchical data like org charts, category trees, or graph structures.

```sql
-- Find all subordinates of a manager (org chart traversal)
WITH RECURSIVE subordinates AS (
    -- Base case: the manager
    SELECT id, name, manager_id, 0 AS depth
    FROM employees
    WHERE id = 42

    UNION ALL

    -- Recursive case: employees who report to someone in our result set
    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.id
    WHERE s.depth < 10  -- safety limit to prevent infinite recursion
)
SELECT * FROM subordinates ORDER BY depth, name;
```

**Real-world use case:** Displaying a category breadcrumb (Electronics > Phones > Smartphones) by walking up the category tree from a leaf node.

---

## Subqueries

Subqueries are queries nested inside other queries. Use them when a CTE feels too heavyweight for a simple filter.

### Scalar subquery (returns one value)

```sql
-- Users who have spent more than average
SELECT name, total_spent
FROM user_stats
WHERE total_spent > (SELECT AVG(total_spent) FROM user_stats);
```

### EXISTS (correlated subquery)

```sql
-- Users who have at least one order in the last 30 days
SELECT u.name, u.email
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
    AND o.created_at > NOW() - INTERVAL '30 days'
);
```

**Performance note:** `EXISTS` stops scanning as soon as it finds one match. It is generally faster than `IN (SELECT ...)` for large subquery results.

### Lateral joins

A lateral join lets the subquery reference columns from the preceding table. Think of it as a "for each row" subquery.

```sql
-- For each user, get their 3 most recent orders
SELECT u.name, recent.id AS order_id, recent.total, recent.created_at
FROM users u
CROSS JOIN LATERAL (
    SELECT o.id, o.total, o.created_at
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 3
) recent;
```

---

## Practical Business Problems

### Leaderboard with rank

```sql
WITH user_scores AS (
    SELECT
        u.id,
        u.name,
        u.avatar_url,
        SUM(a.points) AS total_points
    FROM users u
    INNER JOIN activities a ON u.id = a.user_id
    WHERE a.created_at > DATE_TRUNC('month', NOW())
    GROUP BY u.id, u.name, u.avatar_url
)
SELECT
    DENSE_RANK() OVER (ORDER BY total_points DESC) AS rank,
    name,
    avatar_url,
    total_points
FROM user_scores
ORDER BY rank
LIMIT 100;
```

### Cohort retention analysis

```sql
-- What percentage of users who signed up each month are still active N months later?
WITH user_cohorts AS (
    SELECT
        id AS user_id,
        DATE_TRUNC('month', created_at) AS cohort_month
    FROM users
),
user_activity AS (
    SELECT
        user_id,
        DATE_TRUNC('month', created_at) AS activity_month
    FROM orders
)
SELECT
    c.cohort_month,
    COUNT(DISTINCT c.user_id) AS cohort_size,
    COUNT(DISTINCT CASE WHEN a.activity_month = c.cohort_month + INTERVAL '1 month'
        THEN c.user_id END) AS retained_month_1,
    COUNT(DISTINCT CASE WHEN a.activity_month = c.cohort_month + INTERVAL '2 months'
        THEN c.user_id END) AS retained_month_2,
    COUNT(DISTINCT CASE WHEN a.activity_month = c.cohort_month + INTERVAL '3 months'
        THEN c.user_id END) AS retained_month_3
FROM user_cohorts c
LEFT JOIN user_activity a ON c.user_id = a.user_id
GROUP BY c.cohort_month
ORDER BY c.cohort_month;
```

### Gap detection (finding missing data)

```sql
-- Find days with no orders (useful for monitoring/alerting)
WITH date_range AS (
    SELECT generate_series(
        '2024-01-01'::date,
        CURRENT_DATE,
        '1 day'::interval
    )::date AS date
),
daily_orders AS (
    SELECT created_at::date AS date, COUNT(*) AS order_count
    FROM orders
    GROUP BY created_at::date
)
SELECT dr.date
FROM date_range dr
LEFT JOIN daily_orders do ON dr.date = do.date
WHERE do.order_count IS NULL
ORDER BY dr.date;
```

---

## Pitfalls

- **Overusing subqueries where a JOIN suffices.** Correlated subqueries run once per row. A JOIN often produces the same result more efficiently.
- **CTEs as optimization barriers.** In PostgreSQL 12+, CTEs can be inlined by the optimizer. In older versions, CTEs are materialized as optimization fences. If performance matters on older PG, test CTEs vs subqueries.
- **Window functions without PARTITION BY.** If you forget `PARTITION BY`, the window function operates over the entire result set, which may not be what you intend.
- **Not handling NULLs in LAG/LEAD.** The first row has no `LAG` value (it is NULL). Use `COALESCE` or handle NULLs in your application code.
- **Recursive CTEs without depth limits.** A cycle in your data (employee A reports to B, B reports to A) causes infinite recursion. Always add a `WHERE depth < N` guard.
