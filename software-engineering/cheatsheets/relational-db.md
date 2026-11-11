# Relational Databases

## Introduction

Relational databases organize data into structured tables with defined relationships between them. They remain the backbone of most business applications due to strong consistency guarantees (ACID), a mature ecosystem, and the expressive power of SQL. This cheatsheet covers the essential knowledge every backend developer needs — from writing queries to designing schemas that scale.

**Why This Matters:**

- Nearly every web application stores data in a relational database

- Poor schema design causes performance issues that are expensive to fix later

- Understanding indexes is the single biggest lever for query performance

- ORM misuse is the \#1 source of N+1 query bugs in production

------------------------------------------------------------------------

## SQL Fundamentals

### CRUD Operations

``` sql
-- CREATE: Insert new rows
INSERT INTO users (name, email, created_at)
VALUES ('Alice', '[email protected]', NOW());

-- READ: Query existing rows
SELECT id, name, email
FROM users
WHERE is_active = TRUE
ORDER BY created_at DESC
LIMIT 20;

-- UPDATE: Modify existing rows
UPDATE users
SET email = '[email protected]'
WHERE id = 42;

-- DELETE: Remove rows (prefer soft-delete in production)
DELETE FROM users WHERE id = 42;

-- Soft delete pattern (recommended)
UPDATE users SET deleted_at = NOW() WHERE id = 42;
```

### JOINs

JOINs combine rows from two or more tables based on a related column:

``` mermaid
        erDiagram
    users ||--o{ orders : places
    orders ||--|{ order_items : contains
    order_items }o--|| products : references
    users {
        int id PK
        string name
        string email
    }
    orders {
        int id PK
        int user_id FK
        datetime created_at
    }
    order_items {
        int id PK
        int order_id FK
        int product_id FK
        int quantity
    }
    products {
        int id PK
        string name
        decimal price
    }
```

``` sql
-- INNER JOIN: Only matching rows from both tables
SELECT u.name, o.id AS order_id, o.created_at
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: All rows from left table, matching from right (NULL if no match)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- Multi-table JOIN
SELECT u.name, p.name AS product, oi.quantity
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at > '2026-01-01';
```

| JOIN Type | Returns |
|----|----|
| **INNER JOIN** | Only rows with matches in both tables |
| **LEFT JOIN** | All left rows + matching right rows (NULL if no match) |
| **RIGHT JOIN** | All right rows + matching left rows (NULL if no match) |
| **FULL OUTER JOIN** | All rows from both tables (NULL where no match) |
| **CROSS JOIN** | Cartesian product (every combination) |

### Aggregation and Grouping

``` sql
-- COUNT, SUM, AVG, MIN, MAX
SELECT
    status,
    COUNT(*) AS ticket_count,
    AVG(resolution_hours) AS avg_hours
FROM tickets
GROUP BY status
HAVING COUNT(*) > 5
ORDER BY ticket_count DESC;
```

### Subqueries and CTEs

``` sql
-- Subquery: Find users who have placed orders
SELECT name FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- CTE (Common Table Expression): More readable alternative
WITH active_users AS (
    SELECT DISTINCT user_id
    FROM orders
    WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT u.name, u.email
FROM users u
JOIN active_users au ON u.id = au.user_id;
```

### Window Functions

Window functions perform calculations across a set of rows related to the current row without collapsing them:

``` sql
-- ROW_NUMBER: Rank users by order count
SELECT
    u.name,
    COUNT(o.id) AS order_count,
    ROW_NUMBER() OVER (ORDER BY COUNT(o.id) DESC) AS rank
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- LAG/LEAD: Compare with previous/next row
SELECT
    date,
    revenue,
    revenue - LAG(revenue) OVER (ORDER BY date) AS daily_change
FROM daily_sales;
```

------------------------------------------------------------------------

## Database Normalization

Normalization organizes tables to reduce redundancy and prevent update anomalies. Apply normal forms progressively:

### First Normal Form (1NF)

**Rule:** Each column contains only atomic (indivisible) values. No repeating groups.

```
BAD (violates 1NF):
| student_id | name  | phone_numbers          |
|------------|-------|------------------------|
| 1          | Alice | 0901234567,0912345678  |  ← multi-value

GOOD (1NF):
| student_id | name  | phone_number |
|------------|-------|--------------|
| 1          | Alice | 0901234567   |
| 1          | Alice | 0912345678   |
```

### Second Normal Form (2NF)

**Rule:** 1NF + every non-key column depends on the **entire** primary key (no partial dependencies).

```
BAD (violates 2NF — composite key {student_id, course_id}):
| student_id | course_id | student_name | grade |
← student_name depends only on student_id, not the full key

GOOD (2NF): Split into two tables:
students: | student_id | student_name |
grades:   | student_id | course_id | grade |
```

### Third Normal Form (3NF)

**Rule:** 2NF + no transitive dependencies (non-key columns depend only on the primary key, not on other non-key columns).

```
BAD (violates 3NF):
| employee_id | department_id | department_name |
← department_name depends on department_id, not employee_id

GOOD (3NF): Split into two tables:
employees:   | employee_id | department_id |
departments: | department_id | department_name |
```

### When to Denormalize

Normalization optimizes for writes and data integrity. Sometimes you trade that for read performance:

| Scenario | Denormalization Strategy |
|----|----|
| Dashboard queries joining 5+ tables | Materialized views or summary tables |
| High-traffic read endpoints | Cache frequently-joined data in a single table |
| Reporting/analytics | Star schema (fact + dimension tables) |
| Full-text search | Denormalized search index (Elasticsearch, pg_trgm) |

**Start normalized (3NF), denormalize only when you have measured performance problems.** Premature denormalization causes data inconsistency bugs that are hard to debug.

------------------------------------------------------------------------

## Indexing Strategies

Indexes are the most impactful tool for query performance. They trade write speed and storage for dramatically faster reads.

### How Indexes Work

Without an index, the database performs a **sequential scan** (reads every row). With an index, it performs an **index scan** (jumps directly to matching rows).

``` sql
-- Without index: Sequential scan on 1M rows → slow
SELECT * FROM users WHERE email = '[email protected]';

-- With index: Index lookup → fast
CREATE INDEX idx_users_email ON users (email);
```

### B-Tree Indexes (Default)

The most common index type. Supports equality (`=`) and range queries (`<`, `>`, `BETWEEN`, `LIKE`` ``'prefix%'`).

``` sql
-- Single column index
CREATE INDEX idx_orders_created ON orders (created_at);

-- Composite index (column order matters!)
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- Unique index (also enforces uniqueness)
CREATE UNIQUE INDEX idx_users_email ON users (email);
```

**Composite index column order rule:** The index on `(A,`` ``B,`` ``C)` supports queries filtering on:

- `A` alone

- `A` and `B`

- `A`, `B`, and `C`

- But NOT `B` alone or `C` alone (leftmost prefix rule)

### Hash Indexes

Optimized for exact equality lookups only. Not useful for range queries.

``` sql
-- PostgreSQL hash index
CREATE INDEX idx_sessions_token ON sessions USING hash (token);

-- Good for: WHERE token = 'abc123'
-- Bad for:  WHERE token LIKE 'abc%' or ORDER BY token
```

### Partial Indexes

Index only a subset of rows — smaller and faster:

``` sql
-- Only index active users (skip millions of inactive rows)
CREATE INDEX idx_active_users ON users (email) WHERE is_active = TRUE;

-- Only index pending orders
CREATE INDEX idx_pending_orders ON orders (created_at) WHERE status = 'pending';
```

### GIN Indexes (Full-Text and JSONB)

``` sql
-- Full-text search
CREATE INDEX idx_articles_search ON articles USING gin (to_tsvector('english', title || ' ' || body));

-- JSONB field queries
CREATE INDEX idx_metadata ON events USING gin (metadata);
```

### Index Anti-Patterns

| Anti-Pattern | Problem | Fix |
|----|----|----|
| Index every column | Slows down INSERT/UPDATE, wastes storage | Index only columns used in WHERE, JOIN, ORDER BY |
| Missing composite index | DB scans one index then filters in memory | Create composite index matching your query pattern |
| Function on indexed column | `WHERE`` ``LOWER(email)`` ``=`` ``...` bypasses index | Create expression index: `CREATE`` ``INDEX`` ``ON`` ``users`` ``(LOWER(email))` |
| Unused indexes | Wastes write performance and storage | Monitor with `pg_stat_user_indexes`, drop unused |

### Analyzing Query Performance

``` sql
-- EXPLAIN shows the query plan
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 10;

-- Look for:
-- ✓ Index Scan or Index Only Scan → good
-- ✗ Seq Scan on large tables → needs index
-- ✗ Sort → consider adding ORDER BY column to index
```

------------------------------------------------------------------------

## ORM Patterns (SQLAlchemy)

ORMs map database tables to Python classes, providing a higher-level API for database operations.

### Model Definition

``` python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, Boolean
from sqlalchemy.orm import relationship, DeclarativeBase
from datetime import datetime


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), unique=True, nullable=False, index=True)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationship: one user has many orders
    orders = relationship("Order", back_populates="user")


class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    total = Column(Integer, nullable=False)
    status = Column(String(20), default="pending")
    created_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("User", back_populates="orders")
```

### The N+1 Query Problem

The most common ORM performance bug:

``` python
# BAD: N+1 queries (1 query for users + N queries for orders)
users = session.query(User).all()
for user in users:
    print(user.orders)  # Each access triggers a separate query!

# GOOD: Eager loading with joinedload (1 query total)
from sqlalchemy.orm import joinedload

users = session.query(User).options(joinedload(User.orders)).all()
for user in users:
    print(user.orders)  # Already loaded, no extra queries
```

### Common ORM Operations

``` python
from sqlalchemy import select

# Create
new_user = User(name="Alice", email="[email protected]")
session.add(new_user)
session.commit()

# Read with filtering
stmt = select(User).where(User.is_active == True).order_by(User.created_at.desc())
users = session.execute(stmt).scalars().all()

# Update
user = session.get(User, 42)
user.email = "[email protected]"
session.commit()

# Delete
session.delete(user)
session.commit()

# Pagination
stmt = (
    select(User)
    .where(User.is_active == True)
    .offset(20)
    .limit(10)
)
```

------------------------------------------------------------------------

## Schema Design Best Practices

### Naming Conventions

| Element         | Convention              | Example                  |
|-----------------|-------------------------|--------------------------|
| Tables          | Plural, snake_case      | `order_items`            |
| Columns         | Singular, snake_case    | `created_at`             |
| Primary keys    | `id`                    | `users.id`               |
| Foreign keys    | `<singular_table>_id`   | `orders.user_id`         |
| Indexes         | `idx_<table>_<columns>` | `idx_orders_user_status` |
| Boolean columns | `is_` or `has_` prefix  | `is_active`, `has_paid`  |

### Essential Columns for Every Table

``` sql
CREATE TABLE example (
    id SERIAL PRIMARY KEY,
    -- ... business columns ...
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

Auto-update `updated_at` with a PostgreSQL trigger:

```
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON example
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### Soft Delete Pattern

``` sql
-- Add deleted_at column (NULL = not deleted)
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;

-- Query only active records
SELECT * FROM users WHERE deleted_at IS NULL;

-- Partial index for performance
CREATE INDEX idx_users_active ON users (email) WHERE deleted_at IS NULL;
```

------------------------------------------------------------------------

## PostgreSQL-Specific Tips

### Useful Data Types

| Type | Use Case | Example |
|----|----|----|
| `UUID` | Distributed primary keys | `gen_random_uuid()` |
| `JSONB` | Semi-structured data, flexible metadata | `metadata`` ``JSONB` |
| `TEXT[]` | Tags, categories | `tags`` ``TEXT[]` |
| `TIMESTAMPTZ` | Always use over `TIMESTAMP` | `created_at`` ``TIMESTAMPTZ` |
| `ENUM` | Fixed set of values | `CREATE`` ``TYPE`` ``status`` ``AS`` ``ENUM`` ``(...)` |

### Performance Tips

``` sql
-- Use TIMESTAMPTZ (timezone-aware), not TIMESTAMP
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Use EXPLAIN ANALYZE to understand query performance
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 42;

-- Vacuum and analyze regularly (usually automatic)
VACUUM ANALYZE orders;
```

------------------------------------------------------------------------

## Summary

| Topic | Key Takeaway |
|----|----|
| **SQL** | Master JOINs, CTEs, and window functions — they solve 90% of query needs |
| **Normalization** | Start at 3NF, denormalize only with measured evidence |
| **Indexing** | Index columns in WHERE, JOIN, ORDER BY. Use composite indexes matching query patterns |
| **ORM** | Use eager loading to avoid N+1. Use raw SQL for complex analytics |
| **Schema** | Always include `created_at`/`updated_at`. Use soft deletes. Use `TIMESTAMPTZ` |

------------------------------------------------------------------------

## References

1.  [PostgreSQL Documentation](https://www.postgresql.org/docs/)

2.  [Use The Index, Luke — SQL Indexing Explained](https://use-the-index-luke.com/)

3.  [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/en/20/)

4.  [Database Normalization Explained](https://www.guru99.com/database-normalization.html)

5.  [PostgreSQL Performance Tips](https://wiki.postgresql.org/wiki/Performance_Optimization)

## Practice

### Exercise 1: Schema Design

Design a database schema for a **blog platform** with the following requirements:

- Users can create posts

- Posts can have multiple tags (many-to-many)

- Users can comment on posts

- Comments support nesting (replies to comments)

**Deliverables:**

1.  Draw an ER diagram (use Mermaid)

2.  Write the `CREATE`` ``TABLE` statements

3.  Identify which columns need indexes and why

4.  What normal form is your design in? Justify.

### Exercise 2: Query Optimization

Given this slow query on a table with 5 million rows:

``` sql
SELECT u.name, COUNT(o.id) as order_count, SUM(o.total) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.is_active = TRUE
  AND o.created_at > '2025-01-01'
GROUP BY u.id, u.name
HAVING SUM(o.total) > 1000
ORDER BY total_spent DESC
LIMIT 20;
```

**Tasks:**

1.  Run `EXPLAIN`` ``ANALYZE` and identify bottlenecks

2.  Create appropriate indexes to optimize this query

3.  Rewrite the query using a CTE if it improves readability

4.  Measure before/after execution time

### Exercise 3: N+1 Detection

Given this SQLAlchemy code, identify the N+1 problem and fix it:

``` python
users = session.query(User).filter(User.is_active == True).all()
result = []
for user in users:
    result.append({
        "name": user.name,
        "order_count": len(user.orders),
        "latest_order": user.orders[-1].created_at if user.orders else None,
    })
```

## Review Questions

1.  **What is the difference between `INNER`` ``JOIN` and `LEFT`` ``JOIN`? When would you use each?**

    - *Hint*: Think about what happens when there is no matching row in the joined table.

2.  **Explain the three normal forms (1NF, 2NF, 3NF) in your own words. Give an example of a table that violates 2NF.**

    - *Hint*: Focus on what each form eliminates (repeating groups, partial dependencies, transitive dependencies).

3.  **You have a table with 10 million rows. A query filtering by `status` and `created_at` is slow. What index would you create and why?**

    - *Hint*: Consider composite index column order and the leftmost prefix rule.

4.  **What is the N+1 query problem? How do you detect and fix it in SQLAlchemy?**

    - *Hint*: Think about lazy loading vs eager loading.

5.  **When would you choose to denormalize a database? What are the trade-offs?**

    - *Hint*: Consider read performance vs write complexity and data consistency.

6.  **What is the difference between a B-Tree index and a Hash index? When would you use each?**

    - *Hint*: Think about the types of queries each supports (equality, range, ordering).

7.  **Why should you use `TIMESTAMPTZ` instead of `TIMESTAMP` in PostgreSQL?**

    - *Hint*: Consider what happens when your application serves users in different time zones.

8.  **What does `EXPLAIN`` ``ANALYZE` tell you that `EXPLAIN` alone does not?**

    - *Hint*: One shows the plan, the other shows plan + actual execution statistics.
