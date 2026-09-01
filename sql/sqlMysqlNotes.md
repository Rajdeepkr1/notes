# SQL & MySQL — Deep Dive Roadmap

We'll go from fundamentals → query internals → schema design → performance → transactions → replication → security → production → interview problems.

---

## 1. SQL Fundamentals

**Definition:** a relational database organizes data into **tables** — grid-like structures of rows and columns — related to one another through shared key values, queried and manipulated using **SQL** (Structured Query Language), a declarative language where you describe *what* data you want, not the steps to retrieve it.

**Tables, rows, columns — Definition:** a **table** represents one type of entity (e.g. `users`); a **row** (or "tuple"/"record") is one instance of that entity; a **column** (or "field"/"attribute") is one named, typed property shared by every row in the table.

**SQL sub-languages — Definition:**
- **DDL** (Data Definition Language) — defines/alters structure: `CREATE`, `ALTER`, `DROP`.
- **DML** (Data Manipulation Language) — modifies data: `INSERT`, `UPDATE`, `DELETE`.
- **DQL** (Data Query Language) — retrieves data: `SELECT`.
- **DCL** (Data Control Language) — manages permissions: `GRANT`, `REVOKE`.
- **TCL** (Transaction Control Language) — manages transactions: `COMMIT`, `ROLLBACK`, `SAVEPOINT`.

**Data types — Definition:** each column is declared with a type constraining what values it may hold — numeric (`INT`, `BIGINT`, `DECIMAL`, `FLOAT`), string (`VARCHAR(n)`, `TEXT`, `CHAR(n)`), date/time (`DATE`, `DATETIME`, `TIMESTAMP`), boolean (`BOOLEAN`, stored as `TINYINT(1)` in MySQL), and semi-structured (`JSON`).

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  age INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Primary keys & foreign keys — Definition:** a **primary key** uniquely identifies each row in a table (and is automatically indexed); a **foreign key** is a column referencing another table's primary key, enforcing that the referenced row actually exists — the mechanism that expresses relationships between tables at the database level.

```sql
CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**`NULL` semantics — Definition:** `NULL` represents the *absence* of a value, not zero or an empty string; it does not equal anything, including itself (`NULL = NULL` evaluates to `NULL`/unknown, not `TRUE`) — comparisons against `NULL` must use `IS NULL`/`IS NOT NULL`.

**Basic CRUD:**

```sql
SELECT * FROM users WHERE age > 18;
INSERT INTO users (email, age) VALUES ('a@b.com', 30);
UPDATE users SET age = 31 WHERE id = 1;
DELETE FROM users WHERE id = 1;
```

---

## 2. Querying Data

**`SELECT` clause order vs execution order — Definition:** SQL is *written* in the order `SELECT` → `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `ORDER BY` → `LIMIT`, but the database *executes* it in a different logical order: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT` — which is why, for example, you can't reference a `SELECT`-defined alias inside a `WHERE` clause (the `WHERE` runs before `SELECT` conceptually), but you *can* reference it in `ORDER BY` (which runs after).

**`WHERE` filtering:**

```sql
SELECT * FROM orders WHERE status = 'completed' AND total > 100;
```

**Comparison & logical operators** — `=`, `<>`/`!=`, `<`, `>`, `<=`, `>=`, `AND`, `OR`, `NOT`.

**`LIKE` / pattern matching — Definition:** matches strings against a pattern using `%` (any sequence of characters) and `_` (any single character).

```sql
SELECT * FROM users WHERE email LIKE '%@gmail.com';
```

**`IN` / `BETWEEN` / `IS NULL`:**

```sql
SELECT * FROM orders WHERE status IN ('pending', 'processing');
SELECT * FROM orders WHERE total BETWEEN 50 AND 200;
SELECT * FROM users WHERE deleted_at IS NULL;
```

**`ORDER BY` / `LIMIT` / `OFFSET`:**

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10 OFFSET 20;
```

**`DISTINCT` — Definition:** removes duplicate rows from the result set, evaluated across all selected columns together (not per-column).

**Aliasing (`AS`)** — renames a column or table for readability within the query (and its output), e.g. `SELECT COUNT(*) AS total_orders FROM orders`.

---

## 3. Joins

**Definition:** a join combines rows from two or more tables based on a related column between them — the fundamental relational operation for reassembling normalized (split-apart) data back into a single result set.

**`INNER JOIN` — Definition:** returns only rows that have a matching row in *both* tables — non-matching rows on either side are excluded entirely.

```sql
SELECT orders.id, users.email
FROM orders
INNER JOIN users ON orders.user_id = users.id;
```

**`LEFT JOIN` / `RIGHT JOIN` — Definition:** a **LEFT JOIN** returns *all* rows from the left table, plus matching rows from the right table (with `NULL`s for right-side columns when there's no match); **RIGHT JOIN** is the mirror image — in practice, `RIGHT JOIN` is rarely used, since any `RIGHT JOIN` can be rewritten as an equivalent `LEFT JOIN` by swapping table order.

```sql
-- users with no orders still appear, with NULL order columns
SELECT users.email, orders.id
FROM users
LEFT JOIN orders ON orders.user_id = users.id;
```

**`FULL OUTER JOIN` — Definition:** returns all rows from both tables, matched where possible, with `NULL`s on whichever side lacks a match. **MySQL has no native `FULL OUTER JOIN`** — it's emulated with a `LEFT JOIN UNION RIGHT JOIN` (or `UNION ALL` with appropriate exclusion).

```sql
SELECT * FROM a LEFT JOIN b ON a.id = b.a_id
UNION
SELECT * FROM a RIGHT JOIN b ON a.id = b.a_id;
```

**`CROSS JOIN` — Definition:** returns the Cartesian product of both tables — every row from the first table paired with every row from the second — used deliberately (generating combinations) but a common accidental bug when a `JOIN` condition is forgotten.

**Self joins — Definition:** a table joined to itself, typically via aliases, used for hierarchical or comparative relationships within one table (e.g. an `employees` table with a `manager_id` referencing another row in the same table).

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**Joining vs subqueries** — a join and an equivalent subquery often produce the same result, but joins let the optimizer consider the whole query holistically (often faster, especially with good indexes), while correlated subqueries (section 5) can force row-by-row re-evaluation — generally prefer a join unless the query logic genuinely needs a subquery's semantics (e.g. `EXISTS`).

---

## 4. Aggregation

**`GROUP BY` — Definition:** collapses multiple rows sharing the same value(s) in specified column(s) into a single summary row per group, computed via aggregate functions.

```sql
SELECT status, COUNT(*) AS count, SUM(total) AS revenue
FROM orders
GROUP BY status;
```

**Aggregate functions — Definition:** `COUNT()` (number of rows, or non-null values of a specific column), `SUM()`, `AVG()`, `MIN()`, `MAX()` — each collapses a set of values into a single value per group.

**`HAVING` vs `WHERE` — Definition:** `WHERE` filters individual rows *before* grouping; `HAVING` filters *groups* after aggregation — a condition on an aggregate function (`COUNT(*) > 5`) must go in `HAVING`, since `WHERE` runs before aggregates are even computed (recall the logical execution order from section 2).

```sql
SELECT customer_id, COUNT(*) AS order_count
FROM orders
WHERE status = 'completed'      -- filters rows first
GROUP BY customer_id
HAVING COUNT(*) > 5;             -- filters groups after aggregation
```

**`GROUP BY` with multiple columns** — groups are formed by the unique combination of *all* listed columns, not each independently.

**`ROLLUP` — Definition:** an extension to `GROUP BY` that adds extra summary rows for each level of subtotal, plus a grand total — e.g. `GROUP BY region, product WITH ROLLUP` adds a subtotal row per region and one grand-total row.

**Common aggregation pitfalls** — selecting a non-aggregated, non-grouped column (MySQL's default SQL mode may allow this and return an arbitrary value from the group — almost always a bug); forgetting `HAVING` vs `WHERE`; `COUNT(column)` ignoring `NULL`s while `COUNT(*)` counts all rows.

---

## 5. Subqueries & CTEs

**Scalar subqueries — Definition:** a subquery that returns exactly one row and one column, usable anywhere a single value is expected.

```sql
SELECT name, (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.id) AS order_count
FROM users;
```

**Subqueries in `WHERE`/`FROM`/`SELECT`** — a subquery can filter (`WHERE id IN (SELECT ...)`), act as a derived table (`FROM (SELECT ...) AS t`), or compute a per-row value (as above).

**Correlated vs non-correlated — Definition:** a **non-correlated** subquery is independent of the outer query and can run on its own; a **correlated** subquery references a column from the outer query, so it conceptually re-executes once per outer row — often slower unless the optimizer can rewrite it into a join.

```sql
-- correlated: references outer table `u`
SELECT * FROM users u WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

**`EXISTS` vs `IN` — Definition:** both check for matching rows in a subquery; `EXISTS` stops at the first match (often more efficient, and correctly handles `NULL`s in the subquery's result, unlike `IN`, which can behave unexpectedly if the subquery returns any `NULL`).

**Common Table Expressions (`WITH`) — Definition:** a named, temporary result set defined at the start of a query, referenced later in that query — improves readability by breaking a complex query into named, logical steps, and can be referenced multiple times without repeating the subquery.

```sql
WITH high_value_orders AS (
  SELECT * FROM orders WHERE total > 1000
)
SELECT user_id, COUNT(*) FROM high_value_orders GROUP BY user_id;
```

**Recursive CTEs — Definition:** a CTE that references itself, used for traversing hierarchical or graph-like data (org charts, category trees) — an initial "anchor" query is unioned with a "recursive" query that repeatedly joins back to the CTE until no new rows are produced.

```sql
WITH RECURSIVE org_chart AS (
  SELECT id, name, manager_id, 1 AS depth FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id, oc.depth + 1
  FROM employees e JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart;
```

---

## 6. Window Functions

**Definition:** a window function performs a calculation across a set of rows related to the current row (its "window"), *without* collapsing those rows into a single output row the way `GROUP BY` does — each input row still produces one output row, just enriched with an aggregate/ranking value computed over its window.

**`OVER()` clause — Definition:** the syntax that turns an aggregate/ranking function into a window function, defining the window (partition, order, frame) it operates over.

```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
-- every employee row is kept, each annotated with their department's average
```

**`PARTITION BY` — Definition:** divides rows into groups (like `GROUP BY`, but without collapsing rows) — the window function is computed independently within each partition.

**`ROW_NUMBER()` / `RANK()` / `DENSE_RANK()` — Definition:** assign a sequential position to each row within its window, ordered by `ORDER BY` inside `OVER()`. `ROW_NUMBER()` always gives unique, sequential numbers (arbitrary tie-breaking); `RANK()` gives tied rows the same rank but leaves a gap afterward (1, 2, 2, 4); `DENSE_RANK()` gives tied rows the same rank with no gap (1, 2, 2, 3).

```sql
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;
```

**`LAG()` / `LEAD()` — Definition:** access a value from a preceding (`LAG`) or following (`LEAD`) row within the same window, without a self-join — commonly used for period-over-period comparisons.

```sql
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue
FROM monthly_sales;
```

**Running totals & moving averages:**

```sql
SELECT order_date, total,
       SUM(total) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

**Frame clauses (`ROWS BETWEEN`) — Definition:** fine-tunes exactly which rows within the partition are included relative to the current row (e.g. "the current row and the 2 preceding it," for a 3-row moving average), rather than the entire partition.

```sql
AVG(total) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

---

## 7. Schema Design & Normalization

**Definition:** normalization is the process of organizing a schema to minimize data redundancy and avoid update anomalies, by progressively applying a series of rules ("normal forms").

- **1NF (First Normal Form)** — every column holds a single, atomic value (no repeating groups or comma-separated lists in one field).
- **2NF** — 1NF, plus every non-key column depends on the *entire* primary key (relevant only for composite keys — eliminates partial dependency).
- **3NF** — 2NF, plus no non-key column depends on another non-key column (eliminates transitive dependency — e.g. storing a `city` derived from a stored `zip_code` on the same table violates 3NF).
- **BCNF (Boyce-Codd Normal Form)** — a stricter version of 3NF handling certain edge cases with overlapping composite candidate keys.

**Denormalization tradeoffs — Definition:** deliberately reintroducing redundancy (duplicating a value across tables, or a computed/cached column) to avoid expensive joins/aggregations on read — the same fundamental read-speed-vs-write-complexity tradeoff covered in the MongoDB notes, just less idiomatic in a normalized-by-default relational world.

**Primary keys vs surrogate keys — Definition:** a **natural key** is an existing, real-world-meaningful attribute unique enough to serve as a primary key (e.g. an email or SSN); a **surrogate key** is an artificial, database-generated identifier (an `AUTO_INCREMENT` integer or a UUID) with no business meaning — surrogate keys are generally preferred, since natural keys can change (a business rule shifts) or turn out to be less unique than assumed.

**Foreign key constraints & referential integrity** — see section 1 & 8; enforced by the database so an `orders.user_id` can never reference a nonexistent user, regardless of which application code path performed the insert.

**Relationship modeling:**

```sql
-- one-to-many: orders.user_id references users.id (FK lives on the "many" side)
-- many-to-many: a junction table
CREATE TABLE post_tags (
  post_id INT,
  tag_id INT,
  PRIMARY KEY (post_id, tag_id),
  FOREIGN KEY (post_id) REFERENCES posts(id),
  FOREIGN KEY (tag_id) REFERENCES tags(id)
);
```

**Designing for query patterns** — like the MongoDB notes' modeling guidance, but inverted: relational schema design *starts* from normalization (correctness/integrity first), then deliberately denormalizes specific hot paths once real query patterns and performance data justify it — rather than MongoDB's "model for reads from the start."

---

## 8. Constraints & Data Integrity

**Definition:** constraints are rules enforced by the database itself on every write, regardless of which application or script performs it — the strongest layer of data-integrity defense, beneath any application-level validation.

```sql
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(50) NOT NULL UNIQUE,       -- NOT NULL + UNIQUE
  price DECIMAL(10,2) CHECK (price >= 0), -- CHECK
  status VARCHAR(20) DEFAULT 'active',    -- DEFAULT
  category_id INT,
  FOREIGN KEY (category_id) REFERENCES categories(id)
    ON DELETE SET NULL ON UPDATE CASCADE  -- referential actions
);
```

- **`NOT NULL`** — rejects any write leaving the column empty.
- **`UNIQUE`** — rejects duplicate values across the column (or column combination) — automatically backed by an index.
- **`CHECK`** — rejects any value failing a boolean expression (supported in MySQL 8.0.16+; silently ignored in earlier versions).
- **`DEFAULT`** — supplies a value automatically when none is provided on insert.
- **`FOREIGN KEY` referential actions — Definition:** `ON DELETE`/`ON UPDATE` specify what happens to a dependent row when the referenced row is deleted/updated — `CASCADE` (propagate the delete/update), `SET NULL` (null out the reference), `RESTRICT`/`NO ACTION` (reject the operation if dependents exist).

**Database vs application-layer enforcement** — application-level validation (Zod, Mongoose-style validators) gives friendlier, more specific error messages and can validate before a write is even attempted; database constraints are the non-bypassable backstop, catching anything that reaches the database through any path (a script, a different service, a manual query) — production systems generally want both, same principle as the MongoDB notes' validation-strategy section.

---

## 9. Indexes

A major deep-dive topic.

**Definition:** an index is an auxiliary data structure — in InnoDB, a **B-tree** — that the database maintains alongside a table's data, allowing it to find rows matching a condition without scanning every row.

**Clustered vs non-clustered indexes — Definition:** a **clustered index** determines the *physical storage order* of the table's rows (there can be only one per table, since data can only be physically sorted one way); a **non-clustered (secondary) index** is a separate structure mapping indexed-column values to a pointer back to the row's location.

**Primary key & the clustered index in InnoDB — Definition:** InnoDB always stores table data physically ordered by its primary key (the primary key **is** the clustered index) — every secondary index in InnoDB stores the primary key value (not a raw disk pointer) as its "pointer" back to the full row, which is why a small, sequential (e.g. `AUTO_INCREMENT`) primary key is generally preferred: a large or random primary key (like a UUID) bloats every secondary index and causes expensive page splits/fragmentation from random insert order.

**Composite indexes & column order — Definition:** an index on multiple columns is usable for queries filtering on a **left-prefix** of those columns, in the order they were declared — an index on `(last_name, first_name)` speeds up `WHERE last_name = ?` and `WHERE last_name = ? AND first_name = ?`, but **not** `WHERE first_name = ?` alone.

```sql
CREATE INDEX idx_name ON employees (last_name, first_name);
```

**Covering indexes — Definition:** an index that contains *every* column a query needs (in its filter, sort, and selected columns), letting the query be satisfied entirely from the index without touching the actual table data — the fastest possible query, same concept as MongoDB's covered queries.

**Unique indexes** — see section 8; a `UNIQUE` constraint is implemented as a unique index, enforcing both uniqueness and providing fast lookup.

**Full-text indexes — Definition:** a specialized index (`FULLTEXT`) supporting natural-language and boolean text search across `CHAR`/`VARCHAR`/`TEXT` columns — basic search capability, not a replacement for a dedicated search engine at scale.

**When indexes hurt** — every index speeds up matching reads but adds overhead to every `INSERT`/`UPDATE`/`DELETE` (each write must also update every affected index), and consumes disk/memory — indexing every column "just in case" is a common anti-pattern.

**`EXPLAIN` — Definition:** shows the query execution plan MySQL's optimizer chose — which indexes (if any) are used, the estimated number of rows examined, and the join order — the primary diagnostic tool for understanding and fixing a slow query.

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 5 AND status = 'completed';
```

Key columns in the output: `type` (access method — `const`/`ref`/`range` are good, `ALL` means a full table scan), `key` (which index, if any, was used), `rows` (estimated rows examined — lower is better), `Extra` (`Using index` = covering index; `Using filesort`/`Using temporary` often signal room for optimization).

---

## 10. Transactions & ACID

**Definition:** a transaction is a sequence of one or more SQL operations executed as a single logical unit of work, guaranteed by the database to uphold the **ACID** properties.

- **Atomicity** — all operations in the transaction succeed together, or none of them take effect at all (an error triggers a full rollback).
- **Consistency** — a transaction moves the database from one valid state to another, never violating declared constraints/rules.
- **Isolation** — concurrently-running transactions don't see each other's uncommitted intermediate state (the exact degree of isolation is configurable — see isolation levels below).
- **Durability** — once a transaction commits, its effects survive even a subsequent crash (via the redo log, section 13).

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- or ROLLBACK on error
```

**Savepoints — Definition:** a named point within a transaction that a later `ROLLBACK TO SAVEPOINT` can revert to, without rolling back the entire transaction — useful for partial error recovery within a larger multi-step transaction.

**Isolation levels — Definition:** control how much of one transaction's in-progress changes another concurrent transaction can observe, trading consistency guarantees against concurrency/performance:

| Level | Dirty reads | Non-repeatable reads | Phantom reads |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read (MySQL/InnoDB default) | Prevented | Prevented | Prevented* |
| Serializable | Prevented | Prevented | Prevented |

*InnoDB's Repeatable Read prevents most phantom reads via its MVCC snapshot mechanism, going further than the SQL standard technically requires at that level.

**Anomalies — Definition:**
- **Dirty read** — reading another transaction's *uncommitted* changes, which might later be rolled back.
- **Non-repeatable read** — re-reading the same row twice within one transaction and getting different values, because another transaction committed a change in between.
- **Phantom read** — re-running the same range query twice within one transaction and getting a different *set* of rows, because another transaction inserted/deleted matching rows in between.

**MVCC (Multi-Version Concurrency Control) — Definition:** InnoDB's mechanism for providing high-isolation reads without blocking writers — instead of locking rows for reads, each transaction sees a consistent **snapshot** of the data as of its start (or each statement's start, depending on isolation level), reconstructed from row version history (via the undo log) — readers never block writers, and writers never block readers, under InnoDB's default configuration.

---

## 11. Locking & Concurrency

**Row-level vs table-level locking — Definition:** InnoDB locks at the **row level** by default (only the specific rows a transaction touches are locked, allowing high concurrency); MyISAM (legacy) only supports **table-level** locking (an entire table is locked for any write) — one of the core reasons InnoDB is the standard choice today.

**Shared vs exclusive locks — Definition:** a **shared (S) lock** allows other transactions to also acquire a shared lock on the same row (for reading) but blocks any exclusive lock; an **exclusive (X) lock** (required for writes) blocks any other lock, shared or exclusive, on that row.

**Deadlocks — Definition:** occur when two transactions each hold a lock the other needs, and each is waiting for the other to release — InnoDB automatically detects deadlocks and resolves them by **rolling back one of the transactions** (the one that did less work), returning a deadlock error to that transaction's client, which should retry.

**`SELECT ... FOR UPDATE` — Definition:** explicitly acquires an exclusive row lock on the selected rows within a transaction, preventing other transactions from modifying (or, depending on isolation level, even reading with their own `FOR UPDATE`) those rows until this transaction commits — used to safely read-then-update a value (e.g. checking and decrementing inventory) without a race condition.

```sql
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

**Optimistic vs pessimistic locking — Definition:** **pessimistic** locking (like `FOR UPDATE` above) assumes conflicts are likely and locks proactively; **optimistic** locking assumes conflicts are rare — it proceeds without locking, then checks (typically via a `version` column) at commit time whether the row changed underneath it, retrying if so — better throughput under low contention, at the cost of needing explicit retry logic in the application.

```sql
UPDATE products SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;  -- 0 rows affected = someone else updated it first; retry
```

**Lock contention troubleshooting** — `SHOW ENGINE INNODB STATUS` and the `performance_schema` expose current locks/waiting transactions, used to diagnose which queries are blocking each other in production.

---

## 12. Views, Stored Procedures, Functions, Triggers

**Views — Definition:** a named, stored `SELECT` query that behaves like a virtual table — querying it re-runs the underlying query each time, useful for simplifying repeated complex queries or restricting which columns/rows a given user can see.

```sql
CREATE VIEW active_users AS
SELECT id, email FROM users WHERE deleted_at IS NULL;

SELECT * FROM active_users;
```

**Updatable views** — a view can sometimes be written through (`INSERT`/`UPDATE` against the view, which passes through to the underlying table) if it's simple enough (single table, no aggregation/`GROUP BY`/`DISTINCT`) for MySQL to unambiguously map the operation back.

**Materialized view concept** — MySQL has no native materialized views (a view whose result is physically stored and periodically refreshed, unlike a regular view which re-executes every time); it's commonly emulated with a real table populated by a scheduled job or an `INSERT ... SELECT` triggered periodically or via events.

**Stored procedures — Definition:** a named, precompiled block of SQL (with parameters, control flow) stored and executed *inside* the database server, callable via `CALL`.

```sql
DELIMITER //
CREATE PROCEDURE GetUserOrders(IN userId INT)
BEGIN
  SELECT * FROM orders WHERE user_id = userId;
END //
DELIMITER ;

CALL GetUserOrders(5);
```

**User-defined functions** — similar to stored procedures, but return a single value and can be used inline within a `SELECT` expression, unlike a procedure.

**Triggers — Definition:** a block of SQL automatically executed by the database in response to an `INSERT`/`UPDATE`/`DELETE` on a specific table, firing `BEFORE` or `AFTER` the triggering event.

```sql
CREATE TRIGGER before_order_insert
BEFORE INSERT ON orders
FOR EACH ROW
SET NEW.created_at = NOW();
```

**When to use (and avoid) database-side logic** — stored procedures/triggers keep logic close to the data and can reduce network round trips, but they're harder to version-control, test, and debug than application code, and spread business logic across two different codebases/languages — most modern teams keep business logic in the application layer and reserve triggers/procedures for narrow, data-integrity-focused concerns (auditing, enforcing invariants that must hold regardless of which client writes the data).

---

## 13. MySQL Storage Engines & Architecture

**InnoDB vs MyISAM — Definition:** **InnoDB** (the default since MySQL 5.5) supports transactions, foreign keys, row-level locking, and crash recovery via its redo log — the correct choice for essentially all modern applications. **MyISAM** (legacy) is simpler and was historically faster for read-heavy, non-transactional workloads, but lacks transactions, foreign keys, and crash-safety, and only supports table-level locking.

**InnoDB architecture — Definition:**
- **Buffer pool** — an in-memory cache of table/index data pages, so frequently-accessed data is served from RAM rather than disk — typically the single most impactful tuning parameter (`innodb_buffer_pool_size`) for InnoDB performance.
- **Redo log** — a write-ahead log recording changes *before* they're applied to the actual data files, enabling crash recovery (durability, the "D" in ACID) — on restart after a crash, InnoDB replays the redo log to restore any committed-but-not-yet-flushed changes.
- **Undo log** — stores the *previous* version of rows being modified, used both to roll back an aborted transaction and to serve MVCC snapshot reads for other concurrent transactions (section 10).

**MySQL server architecture (high level):**

```
Client → Connection Handler → Parser → Optimizer → Execution Engine → Storage Engine (InnoDB) → Disk
```

**Connection handling** — each client connection is served by a dedicated thread by default; `max_connections` caps how many concurrent connections the server accepts — exhausting this limit is a common production incident, usually solved with connection pooling on the application side (section 14) rather than raising the limit indefinitely.

**The query cache (deprecated)** — older MySQL versions cached full query results keyed by exact query text, but it was removed in MySQL 8.0: it required a global lock on any write to *any* table the cached query touched, causing severe contention in write-heavy workloads that outweighed its read benefit — modern caching is handled at the application layer instead (e.g. Redis, as covered in the Node.js notes).

---

## 14. Query Optimization & Performance

**`EXPLAIN` / `EXPLAIN ANALYZE` — Definition:** `EXPLAIN` shows the optimizer's *planned* execution strategy without running the query; `EXPLAIN ANALYZE` (MySQL 8.0.18+) actually executes the query and reports *real* timing/row-count data alongside the plan — far more reliable than the optimizer's estimates alone for diagnosing a genuinely slow query.

**Index selection by the optimizer** — the optimizer chooses which index (if any) to use based on estimated selectivity (how much an index narrows the row set) from table statistics — outdated statistics (`ANALYZE TABLE` refreshes them) can cause the optimizer to make a poor choice even when a good index exists.

**Avoiding `SELECT *`** — fetching only needed columns reduces I/O and network transfer, and is a prerequisite for the query to potentially be served by a covering index (section 9), which is impossible if every column must be read from the full row.

**N+1 query problems** — same anti-pattern as covered in the Node.js/MongoDB notes: running one query for a list, then one additional query per row for related data — fixed with a single join or an `IN (...)` batch query instead.

**Slow query log — Definition:** a MySQL feature that logs any query exceeding a configured execution-time threshold (`long_query_time`) — the primary tool for finding *which* queries in a production system are actually slow, rather than guessing.

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- log queries slower than 1 second
```

**Query rewriting techniques** — replacing correlated subqueries with joins, avoiding functions wrapped around indexed columns in `WHERE` (`WHERE YEAR(created_at) = 2024` can't use an index on `created_at`; `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'` can), and avoiding leading-wildcard `LIKE '%term'` (which can't use a standard B-tree index).

**Batch operations:**

```sql
-- one round trip instead of N
INSERT INTO logs (message) VALUES ('a'), ('b'), ('c');
```

**Connection pooling** — see the Node.js notes' section 7; reusing a fixed set of open connections rather than opening one per request/query, since establishing a MySQL connection has real overhead (TCP handshake, authentication).

---

## 15. Replication & High Availability

**Replication fundamentals (binlog-based) — Definition:** MySQL replication works by having a **primary** server record every data-changing statement/row event to its **binary log (binlog)**; one or more **replica** servers connect to the primary, stream the binlog, and re-apply those changes locally, keeping their data in sync.

**Primary-replica replication — Definition:** the standard topology — a single primary accepts all writes; replicas apply the primary's changes and can serve read traffic, offloading read load from the primary.

**Semi-synchronous vs asynchronous replication — Definition:** in **asynchronous** replication (MySQL's default), the primary commits a transaction and returns success to the client *without waiting* for any replica to receive it — simplest and fastest, but a primary crash immediately after commit can lose that transaction before it reached any replica. **Semi-synchronous** replication makes the primary wait for at least one replica to *acknowledge receipt* of the transaction's binlog event before returning success — reduces (but doesn't eliminate) the data-loss window, at some latency cost.

**Read/write splitting — Definition:** application-level (or via a proxy like ProxySQL) routing of write queries to the primary and read queries to replicas — increases read capacity, at the cost of potential **replication lag** (a replica may briefly be behind the primary, so a read immediately following a write might not yet reflect it).

**Failover strategies — Definition:** when a primary fails, a replica must be promoted to the new primary — handled manually, via a script, or automatically by tooling (MySQL Group Replication/InnoDB Cluster, Orchestrator) that also handles redirecting other replicas and application traffic to the new primary.

**MySQL Group Replication / InnoDB Cluster (brief) — Definition:** MySQL's native multi-primary/single-primary replication technology built on a group communication (Paxos-derived) protocol, providing automatic failover and strong consistency guarantees beyond classic binlog replication — MySQL's answer to the same class of problem MongoDB's replica sets solve.

---

## 16. Partitioning & Sharding

**Table partitioning — Definition:** splitting a single large table's data into multiple physical sub-tables ("partitions") *within the same database server*, transparent to queries — used to make maintenance (dropping old data) and certain queries faster, **not** a scaling technique across multiple servers (that's sharding, below).

```sql
CREATE TABLE logs (
  id INT,
  created_at DATE,
  message TEXT
) PARTITION BY RANGE (YEAR(created_at)) (
  PARTITION p2023 VALUES LESS THAN (2024),
  PARTITION p2024 VALUES LESS THAN (2025)
);
```

- **Range partitioning** — by a value range (dates, IDs) — as above; makes bulk-dropping old data (e.g. `DROP PARTITION p2023`) instant, versus a slow `DELETE`.
- **Hash partitioning** — by the hash of a column, for even distribution when there's no natural range.
- **List partitioning** — by an explicit list of discrete values (e.g. by country code).

**When partitioning helps** — very large tables where queries/maintenance naturally align with the partition key (e.g. always querying/archiving by date range) — it doesn't inherently speed up arbitrary queries and adds schema complexity, so it's not a default choice.

**Sharding fundamentals (application-level) — Definition:** unlike partitioning (one server, transparent), sharding splits data across **multiple independent database servers**, with the application (or a routing layer) directing each query to the correct shard — a true horizontal scaling technique, at significant added architectural complexity (cross-shard joins/transactions become hard or impossible).

**Choosing a shard key** — same fundamental concern as the MongoDB notes' shard key section: the key must distribute data/traffic evenly and align with how the application actually queries data, since it's difficult to change after the fact.

**Tradeoffs vs a single large instance** — a well-tuned single primary (with read replicas for read scaling) handles a surprisingly large amount of load; sharding should be a last resort once vertical scaling and read replicas are genuinely insufficient, given how much it complicates application logic and operations.

---

## 17. Security

**User management & privileges — Definition:** MySQL access control is managed via users (identified by `username@host`) and **privileges** granted to them on specific databases/tables/columns.

```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE ON myapp.* TO 'app_user'@'%';
REVOKE DELETE ON myapp.orders FROM 'app_user'@'%';
FLUSH PRIVILEGES;
```

**Principle of least privilege** — an application's database user should have exactly the privileges it needs (rarely `DROP`/`ALTER`, never full admin) — same principle as the AWS IAM notes, applied to database access instead of cloud API access.

**SQL injection & parameterized queries — Definition:** SQL injection is an attack where untrusted input is concatenated directly into a query string and executed as SQL rather than treated as data — prevented by always using **parameterized/prepared statements** (placeholders bound separately from the query text), never string-concatenating user input into SQL.

```sql
-- ❌ vulnerable
`SELECT * FROM users WHERE email = '${email}'`

-- ✅ parameterized (driver-specific placeholder syntax, e.g. mysql2)
connection.query('SELECT * FROM users WHERE email = ?', [email]);
```

**Encryption at rest & in transit** — MySQL supports transparent data encryption for data files at rest, and requires TLS for in-transit encryption between client and server — same concepts as the AWS/MongoDB notes, configured at the database-engine level here.

**Auditing** — MySQL Enterprise Audit (or open-source alternatives/plugins) logs who accessed/modified what, complementing (not replacing) privilege restrictions for compliance and incident investigation.

---

## 18. Backup & Recovery

**Logical backups (`mysqldump`) — Definition:** exports the database as a sequence of SQL statements (`CREATE TABLE`, `INSERT`) that can recreate the data by re-running them — human-readable, portable across MySQL versions, but slower to produce/restore for large databases than a physical backup.

```bash
mysqldump -u root -p mydb > backup.sql
mysql -u root -p mydb < backup.sql   # restore
```

**Physical backups (Percona XtraBackup) — Definition:** copies the actual InnoDB data files directly (hot, without locking the whole database), much faster to back up and restore for large databases than a logical dump, at the cost of being MySQL-version/engine-specific rather than portable SQL text.

**Point-in-time recovery (binlog replay) — Definition:** restoring a full backup taken at some point, then replaying the binary log from that backup's point forward up to a specific desired moment — lets you recover to "right before the accidental `DROP TABLE`" rather than only to the last full backup's timestamp.

**Backup strategy & testing restores** — a backup that has never been restored and verified is not a reliable backup; a sound strategy combines periodic full backups with continuous binlog archiving (for point-in-time recovery), stored off the primary server entirely, with restore drills tested regularly.

---

## 19. Testing

**Testing against a real vs in-memory database** — unlike MongoDB's `mongodb-memory-server`, MySQL doesn't have a widely-used true in-memory equivalent; testing typically uses a real, dedicated test MySQL instance (local, Docker container, or `testcontainers`) rather than an in-memory substitute, since faithfully emulating MySQL's SQL dialect and engine behavior outside real MySQL is impractical.

**Seeding test data** — a SQL script or ORM-driven seed routine populates known fixture rows before tests run, giving predictable assertions.

**Transactional test isolation — Definition:** a common integration-test pattern where each test runs inside a transaction that's **rolled back** at the end, regardless of what the test did — keeps tests independent and the database clean between runs without needing to manually delete data or re-seed from scratch each time.

**Testing migrations** — running each migration (up and down, if reversible) against a copy of production-like data/schema in CI, catching migrations that would fail or lock a large table for too long before they ever reach production.

---

## 20. Production Engineering

**Connection pool sizing — Definition:** the application-side connection pool (section 14) should be sized based on the database server's actual capacity (`max_connections`, available CPU/memory) — too large a pool across many application instances can exhaust the database's connection limit; too small limits application throughput unnecessarily.

**Monitoring key metrics** — slow query rate/count, active connection count vs `max_connections`, replication lag (seconds behind primary on each replica), buffer pool hit ratio, lock wait time — the core set of MySQL health signals worth alerting on in production.

**Migration strategy (schema changes without downtime) — Definition:** on large, high-traffic tables, a naive `ALTER TABLE` can lock the table for the duration of the change; tools like `gh-ost` or `pt-online-schema-change` perform schema changes by creating a shadow table, copying data in the background, and swapping it in with only a brief final lock — the standard approach for zero/near-zero-downtime schema migrations at scale.

**Capacity planning** — projecting data growth and query load ahead of time (informed by the monitoring metrics above) to decide when to add read replicas, upgrade instance size, or begin planning for partitioning/sharding, rather than reacting only after a production incident.

**Common pitfalls & anti-patterns:**
- Missing indexes on foreign key columns and frequently-filtered columns.
- Using `SELECT *` in application code, preventing covering indexes and over-fetching data.
- Large, unbounded transactions holding locks for a long time, blocking other writers.
- Ignoring replication lag when reading from a replica immediately after a write.
- No slow query log / monitoring in production, so performance regressions go unnoticed until they're severe.
- Choosing a large or random primary key (e.g. a raw UUID) on a high-write InnoDB table without considering the clustered-index fragmentation cost (section 9).
