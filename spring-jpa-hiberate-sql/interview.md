# 🚀 Senior Backend Engineer Interview Guide
## Java · Spring Boot · SQL · Hibernate/JPA

> *Written from the perspective of a senior engineer mentoring a mid-level developer preparing for senior-level interviews. Every answer here reflects what interviewers actually want to hear — not textbook definitions, but production-grade thinking.*

---

## 📋 Table of Contents

1. [SQL Section — Senior Level](#-sql-section--senior-level)
   - [Query Optimization & Execution Plans](#1-query-optimization--execution-plans)
   - [Indexing Strategies](#2-indexing-strategies)
   - [Joins & Tricky Scenarios](#3-joins--tricky-scenarios)
   - [Window Functions](#4-window-functions)
   - [Transactions & Isolation Levels](#5-transactions--isolation-levels)
   - [Normalization vs Denormalization](#6-normalization-vs-denormalization)
2. [Hibernate / JPA Section — Senior Level](#-hibernate--jpa-section--senior-level)
   - [Entity Lifecycle & Persistence Context](#1-entity-lifecycle--persistence-context)
   - [Lazy vs Eager Loading & N+1 Problem](#2-lazy-vs-eager-loading--n1-problem)
   - [Caching: 1st Level, 2nd Level, Query Cache](#3-caching-1st-level-2nd-level-query-cache)
   - [Transactions & Propagation](#4-transactions--propagation)
   - [Mapping Strategies](#5-mapping-strategies)
   - [Performance Tuning: Batching & Fetch Joins](#6-performance-tuning-batching--fetch-joins)
3. [Scenario-Based Questions](#-scenario-based-questions)
4. [Comparison & Decision Making](#-comparison--decision-making)
5. [Rapid Revision — Cheat Sheet](#-rapid-revision--cheat-sheet)

---

## 🗄️ SQL Section — Senior Level

---

### 1. Query Optimization & Execution Plans

#### ❓ Interview Question
> *"How do you optimize a slow-running SQL query? Walk me through your thought process."*

#### ✅ What Interviewers Really Want to Hear
They don't want "add an index." They want a systematic diagnostic process — like a surgeon, not a guesser.

#### 📖 Deep Explanation

**Step 1 — Get the execution plan**

```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'PENDING' AND o.created_at > '2024-01-01';

-- MySQL
EXPLAIN FORMAT=JSON SELECT ...;
```

**What to look for in an execution plan:**

| Signal | What It Means | Action |
|--------|--------------|--------|
| `Seq Scan` / `ALL` | Full table scan — no index used | Add index |
| High `rows` estimate vs actual | Stale statistics | Run `ANALYZE` / `UPDATE STATISTICS` |
| `Nested Loop` on large tables | Joining large sets row-by-row | Force Hash Join or add index |
| `Sort` operation | ORDER BY without index | Composite index on sort column |
| `Filter` after scan | WHERE applied after reading rows | Index on filter column |

**Step 2 — Common optimization techniques**

```sql
-- ❌ BAD: Function on indexed column disables the index
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- ✅ GOOD: Range scan — index can be used
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- ❌ BAD: Leading wildcard kills index
SELECT * FROM customers WHERE email LIKE '%@gmail.com';

-- ✅ BETTER: Use full-text index or prefix-only search
SELECT * FROM customers WHERE email LIKE 'john%';

-- ❌ BAD: OR with different columns often causes full scan
SELECT * FROM orders WHERE status = 'PENDING' OR customer_id = 5;

-- ✅ GOOD: UNION approach, each branch uses its own index
SELECT * FROM orders WHERE status = 'PENDING'
UNION ALL
SELECT * FROM orders WHERE customer_id = 5 AND status != 'PENDING';
```

**Step 3 — Statistics & cardinality**

```sql
-- PostgreSQL: refresh statistics
ANALYZE orders;

-- Check table statistics
SELECT relname, n_live_tup, n_dead_tup, last_analyze
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

#### 🏭 Real-World Scenario
*An e-commerce app had a dashboard query timing out at 30+ seconds. The `orders` table had 50M rows. The query was:*

```sql
-- Original slow query
SELECT customer_id, COUNT(*), SUM(total_amount)
FROM orders
WHERE DATE(created_at) = CURDATE()
GROUP BY customer_id;
```

*The fix:*
```sql
-- Step 1: Remove function from column (enables index use)
-- Step 2: Add composite index
CREATE INDEX idx_orders_created_customer
ON orders(created_at, customer_id, total_amount); -- covering index!

-- Rewritten query
SELECT customer_id, COUNT(*), SUM(total_amount)
FROM orders
WHERE created_at >= CURDATE() AND created_at < CURDATE() + INTERVAL 1 DAY
GROUP BY customer_id;
-- Went from 30s → 200ms
```

#### ⚠️ Common Mistakes & Interview Traps
- **"I'll just add an index everywhere"** — Wrong. Indexes slow down writes. Senior engineers think about the read/write ratio.
- **Ignoring row estimates** — If the planner estimates 100 rows but scans 1M, your statistics are stale.
- **Not testing with production-size data** — A query fast on 10K rows can crawl on 10M rows.
- **Forgetting that `SELECT *` can hurt** — Forces the engine to read all columns, prevents covering index usage.

---

### 2. Indexing Strategies

#### ❓ Interview Question
> *"Explain B-Tree indexes, composite indexes, and covering indexes. When would you use each?"*

#### 📖 Deep Explanation

**B-Tree Index (Default — most common)**

Think of it like a phone book. Data is stored in sorted order in a balanced tree. Supports: `=`, `<`, `>`, `BETWEEN`, `LIKE 'prefix%'`, `ORDER BY`.

```sql
-- Simple B-Tree index
CREATE INDEX idx_orders_status ON orders(status);

-- Used by: WHERE status = 'PENDING'
-- NOT used by: WHERE LOWER(status) = 'pending'  ← function invalidates it
```

**Composite Index — Column Order is Everything**

```sql
CREATE INDEX idx_orders_composite ON orders(status, customer_id, created_at);
```

The **"leftmost prefix rule"** — this index can serve queries on:
- `status` alone ✅
- `status + customer_id` ✅
- `status + customer_id + created_at` ✅
- `customer_id` alone ❌ (skipped the leftmost column)
- `created_at` alone ❌

```sql
-- ✅ Uses index (leftmost prefix)
SELECT * FROM orders WHERE status = 'PENDING' AND customer_id = 42;

-- ❌ Does NOT use the composite index
SELECT * FROM orders WHERE customer_id = 42;

-- Rule of thumb: Put HIGH-CARDINALITY or EQUALITY columns first
-- Put RANGE columns (>, <, BETWEEN) last in the composite index
```

**Covering Index — The Secret Weapon**

An index that contains *all* columns the query needs — so the DB never touches the actual table (no "heap fetch").

```sql
-- Query we want to optimize
SELECT customer_id, total_amount, status
FROM orders
WHERE status = 'SHIPPED' AND created_at > '2024-06-01';

-- Covering index: includes all columns in SELECT + WHERE
CREATE INDEX idx_orders_covering
ON orders(status, created_at, customer_id, total_amount);
-- The DB can answer this query ENTIRELY from the index. Zero table reads.
```

**Other Index Types (know these for senior interviews)**

```sql
-- Partial index: only index a subset of rows (PostgreSQL)
CREATE INDEX idx_orders_pending
ON orders(created_at) WHERE status = 'PENDING';
-- Much smaller, much faster for queries that always filter by status='PENDING'

-- Functional/Expression index
CREATE INDEX idx_customers_email_lower
ON customers(LOWER(email));
-- Now LOWER(email) = 'john@example.com' uses the index

-- Hash index: only for exact equality, not range
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
```

#### 🏭 Real-World Scenario
*A fintech app had a query for "all pending transactions in the last 7 days for compliance review." Both `status` and `created_at` were filtered. The team had separate indexes on both columns but the query was still slow.*

```sql
-- The problem: DB was picking only ONE index, then filtering
-- Solution: Composite covering index
CREATE INDEX idx_txn_compliance
ON transactions(status, created_at, amount, account_id);
-- DB now uses one index, reads zero rows from the main table
```

#### ⚠️ Common Mistakes & Interview Traps
- **Too many indexes on write-heavy tables** — Every INSERT/UPDATE must update all indexes. A transaction table with 10 indexes is a performance nightmare.
- **Index on low-cardinality column** — An index on a `boolean` or `gender` column with 2 values is often worse than a full scan (the DB will visit 50% of rows anyway).
- **Not using `INCLUDE` for non-key columns (SQL Server/PostgreSQL)**:
  ```sql
  -- PostgreSQL covering index with INCLUDE
  CREATE INDEX idx_orders_status_incl
  ON orders(status) INCLUDE (total_amount, customer_id);
  ```

---

### 3. Joins & Tricky Scenarios

#### ❓ Interview Question
> *"What's the difference between LEFT JOIN and INNER JOIN? Give me a tricky scenario where using the wrong one causes a bug."*

#### 📖 Deep Explanation

```sql
-- Sample Schema
-- customers(id, name, email)
-- orders(id, customer_id, total_amount, status)

-- INNER JOIN: only rows with a match in BOTH tables
SELECT c.name, o.total_amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;
-- Customers with NO orders disappear from results!

-- LEFT JOIN: all rows from LEFT table, NULLs for unmatched right rows
SELECT c.name, COALESCE(o.total_amount, 0) as amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
-- All customers appear, even those with no orders
```

**The Classic Bug: Filtering a LEFT JOIN Turns it into an INNER JOIN**

```sql
-- ❌ BUG: The WHERE clause on the right table eliminates NULLs
-- This silently becomes an INNER JOIN
SELECT c.name, o.total_amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'PENDING';  -- customers with no orders now excluded!

-- ✅ FIX: Move the filter into the JOIN condition
SELECT c.name, o.total_amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'PENDING';
-- Now customers with no orders still appear (with NULL total_amount)
```

**Self Join — Hierarchical Data**

```sql
-- employees(id, name, manager_id)
-- Find employees and their manager's name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
-- CEO has no manager → LEFT JOIN prevents losing the CEO row
```

**CROSS JOIN — When You Actually Need a Cartesian Product**

```sql
-- Generate a report for every product in every region
SELECT p.name, r.region_name
FROM products p
CROSS JOIN regions r;
-- Legitimate use case: seeding combination tables, report scaffolding
```

**Tricky: Multiple LEFT JOINs Can Inflate Row Counts**

```sql
-- ❌ PROBLEMATIC: if an order has multiple items AND multiple payments
-- This creates a Cartesian product between items and payments!
SELECT o.id, oi.product_name, p.amount
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
LEFT JOIN payments p ON o.id = p.order_id;

-- ✅ FIX: Aggregate first, then join
WITH order_totals AS (
    SELECT order_id, SUM(amount) as total_paid
    FROM payments GROUP BY order_id
)
SELECT o.id, oi.product_name, ot.total_paid
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
LEFT JOIN order_totals ot ON o.id = ot.order_id;
```

#### ⚠️ Common Mistakes & Interview Traps
- **Assuming LEFT JOIN is always "safer"** — It's slower on large tables when you don't need the NULLs. INNER JOIN is faster.
- **Joining on non-indexed columns** — Always ensure JOIN columns are indexed, especially the foreign key side.

---

### 4. Window Functions

#### ❓ Interview Question
> *"Write a query to find the top 3 orders by amount for each customer."*

#### 📖 Deep Explanation

Window functions perform calculations *across a set of rows related to the current row* without collapsing them like GROUP BY does.

```sql
-- ROW_NUMBER: unique rank, no ties
-- RANK: ties get same rank, next rank skips (1,1,3)
-- DENSE_RANK: ties get same rank, no skips (1,1,2)

-- Top 3 orders per customer
WITH ranked_orders AS (
    SELECT
        customer_id,
        order_id,
        total_amount,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id    -- reset counter per customer
            ORDER BY total_amount DESC  -- highest amount = rank 1
        ) AS rn
    FROM orders
)
SELECT customer_id, order_id, total_amount
FROM ranked_orders
WHERE rn <= 3;
```

**Running Totals & Moving Averages**

```sql
-- Running total of daily sales
SELECT
    order_date,
    daily_total,
    SUM(daily_total) OVER (ORDER BY order_date) AS running_total,
    AVG(daily_total) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW  -- 7-day moving average
    ) AS moving_avg_7d
FROM daily_sales;
```

**LAG / LEAD — Compare Rows to Previous/Next**

```sql
-- Month-over-month growth
SELECT
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY month))
        / LAG(revenue, 1) OVER (ORDER BY month) * 100,
    2) AS growth_pct
FROM monthly_revenue;
```

**FIRST_VALUE / LAST_VALUE**

```sql
-- Show each employee's salary vs. the highest salary in their department
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS dept_max_salary
FROM employees;
```

#### ⚠️ Common Mistakes & Interview Traps
- **Using ROW_NUMBER when RANK is needed** — If the interviewer says "top 3 by sales and allow ties," you need RANK or DENSE_RANK.
- **Forgetting PARTITION BY** — Without it, the window is the entire result set.
- **LAST_VALUE gotcha**: LAST_VALUE uses the default frame `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — you must specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` to get the true last value.

---

### 5. Transactions & Isolation Levels

#### ❓ Interview Question
> *"Explain ACID properties. What are phantom reads and how do isolation levels prevent them? What causes a deadlock?"*

#### 📖 Deep Explanation

**ACID Properties**

| Property | What It Means | Real-World Analogy |
|----------|--------------|-------------------|
| **Atomicity** | All operations succeed or all fail (no partial state) | Bank transfer: debit + credit both happen or neither does |
| **Consistency** | DB moves from one valid state to another | Balance never goes negative if constraint exists |
| **Isolation** | Concurrent transactions don't interfere | Two users booking the last seat don't both get it |
| **Durability** | Committed data survives crashes | After "your order is placed," a crash doesn't lose it |

**Isolation Levels & Concurrency Problems**

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|-----------|--------------------|-----------|----|
| READ UNCOMMITTED | ✅ possible | ✅ possible | ✅ possible | Fastest |
| READ COMMITTED | ❌ prevented | ✅ possible | ✅ possible | Fast |
| REPEATABLE READ | ❌ prevented | ❌ prevented | ✅ possible | Medium |
| SERIALIZABLE | ❌ prevented | ❌ prevented | ❌ prevented | Slowest |

```sql
-- Set isolation level
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Dirty Read example (READ UNCOMMITTED):
-- T1: UPDATE accounts SET balance = 500 WHERE id = 1; (not committed)
-- T2: SELECT balance FROM accounts WHERE id = 1; -- reads 500 (dirty!)
-- T1: ROLLBACK;
-- T2 read data that never existed

-- Phantom Read example:
-- T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING'; -- returns 5
-- T2: INSERT INTO orders(status) VALUES('PENDING'); COMMIT;
-- T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING'; -- returns 6! (phantom row appeared)
```

**Deadlocks — How They Happen & Prevention**

```sql
-- Classic deadlock scenario:
-- T1: UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- locks row 1
-- T2: UPDATE accounts SET balance = balance - 100 WHERE id = 2; -- locks row 2
-- T1: UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- waits for T2
-- T2: UPDATE accounts SET balance = balance + 100 WHERE id = 1; -- waits for T1
-- DEADLOCK! Both wait forever.

-- ✅ Prevention Strategy: Always acquire locks in the SAME ORDER
-- T1 and T2 both update id=1 first, then id=2

-- ✅ In Java/Spring:
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // Always lock in ascending ID order
    Long firstId = Math.min(fromId, toId);
    Long secondId = Math.max(fromId, toId);
    Account first = accountRepo.findByIdWithLock(firstId);
    Account second = accountRepo.findByIdWithLock(secondId);
    // ... perform transfer
}
```

**SELECT FOR UPDATE — Pessimistic Locking**

```sql
-- Reserve a seat — prevent double booking
BEGIN;
SELECT * FROM seats WHERE id = 42 AND status = 'AVAILABLE' FOR UPDATE;
-- Now this row is locked. Other transactions block here until we commit.
UPDATE seats SET status = 'RESERVED', user_id = 101 WHERE id = 42;
COMMIT;
```

#### ⚠️ Common Mistakes & Interview Traps
- **"Use SERIALIZABLE for everything"** — This kills concurrency and throughput. Most apps use READ COMMITTED (PostgreSQL default) or REPEATABLE READ (MySQL InnoDB default).
- **Not knowing that PostgreSQL uses MVCC** — PostgreSQL avoids dirty reads at all isolation levels via multi-version concurrency control. This is a senior-level nuance.
- **Deadlock detection** — DBs detect deadlocks automatically and roll back one transaction (the "victim"). Your app must retry on `DeadlockLoserDataAccessException`.

---

### 6. Normalization vs Denormalization

#### ❓ Interview Question
> *"When would you denormalize a database in production? What are the real tradeoffs?"*

#### 📖 Deep Explanation

**Normalization (3NF — Third Normal Form)**

```sql
-- Normalized: orders don't store customer name, just customer_id
CREATE TABLE customers (id INT PRIMARY KEY, name VARCHAR(100), email VARCHAR(200));
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    total DECIMAL(10,2)
);
-- ✅ No redundancy. Update customer name in ONE place.
-- ❌ Every order query requires a JOIN.
```

**Denormalization — When & Why**

```sql
-- Denormalized: store customer_name directly in orders
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);

-- ✅ Benefits:
--   - Faster reads: no JOIN needed for the most common query
--   - Historical accuracy: if customer changes name, old orders still show original name
--   - Works well in read-heavy, write-light scenarios (dashboards, reporting)

-- ❌ Costs:
--   - Data redundancy
--   - Update anomalies: changing customer name means updating all orders
--   - More complex write logic
```

**Real-World Decision Framework**

| Scenario | Recommendation |
|----------|---------------|
| OLTP (banking, e-commerce orders) | Normalize (3NF) |
| Reporting / Analytics | Denormalize or use materialized views |
| Read >> Write (10:1 ratio) | Consider denormalization |
| Historical records (invoices, receipts) | Denormalize by design — snapshot the data |
| Microservices | Each service has its own denormalized view |

```sql
-- Materialized View: best of both worlds
-- Store the expensive JOIN result, refresh periodically
CREATE MATERIALIZED VIEW customer_order_summary AS
SELECT
    c.id,
    c.name,
    COUNT(o.id) AS order_count,
    SUM(o.total) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON c.id = o.id
GROUP BY c.id, c.name;

-- Refresh on schedule or after writes
REFRESH MATERIALIZED VIEW customer_order_summary;
```

#### ⚠️ Common Mistakes & Interview Traps
- **"Always normalize"** — Wrong for senior level. Analytics, event sourcing, and reporting systems often *must* denormalize for performance.
- **Forgetting that event sourcing is intentionally denormalized** — Storing full event payloads is a valid architectural decision.

---

## 🌿 Hibernate / JPA Section — Senior Level

---

### 1. Entity Lifecycle & Persistence Context

#### ❓ Interview Question
> *"Explain the JPA entity lifecycle states and what happens in each transition."*

#### 📖 Deep Explanation

**The 4 Entity States**

```
New/Transient → Managed/Persistent → Detached → Removed
```

```java
@Service
@Transactional
public class OrderService {

    @PersistenceContext
    private EntityManager em;

    public void demonstrateLifecycle() {

        // 1. TRANSIENT — not associated with any persistence context
        Order order = new Order();  // just a Java object, Hibernate doesn't know it exists
        order.setStatus("PENDING");

        // 2. MANAGED/PERSISTENT — tracked by the persistence context
        em.persist(order);  // Now Hibernate tracks every change
        // Any change to 'order' will auto-sync to DB at flush time
        order.setStatus("CONFIRMED"); // No explicit save needed! Dirty checking handles it.

        // 3. DETACHED — was managed, but session closed or explicitly detached
        em.detach(order);
        order.setStatus("SHIPPED"); // Change NOT tracked — will NOT go to DB

        // Re-attach with merge
        Order managedOrder = em.merge(order); // returns a NEW managed instance
        // Important: 'order' is still detached, 'managedOrder' is managed

        // 4. REMOVED — scheduled for deletion
        em.remove(managedOrder);
        // DELETE SQL issued at flush time
    }
}
```

**Dirty Checking — The Magic Behind Hibernate**

```java
@Transactional
public void updateOrderStatus(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    // Hibernate took a snapshot of 'order' when it was loaded

    order.setStatus("SHIPPED");
    // No orderRepository.save() needed!
    // At transaction commit, Hibernate compares current state to snapshot
    // Detects the change → generates UPDATE SQL automatically
}
// Transaction commits → UPDATE orders SET status='SHIPPED' WHERE id=? fired
```

**The Persistence Context as a 1st-Level Cache**

```java
@Transactional
public void showIdentityMap() {
    Order o1 = orderRepository.findById(1L).orElseThrow(); // SELECT query
    Order o2 = orderRepository.findById(1L).orElseThrow(); // NO query! Returns cached instance

    System.out.println(o1 == o2); // true — same Java object reference!
}
```

#### ⚠️ Common Mistakes & Interview Traps
- **Calling `save()` after every field change in a transaction** — Redundant. Dirty checking handles it. It's a code smell that reveals a junior mindset.
- **Modifying a detached entity and expecting it to persist** — Forgetting to re-attach via `merge()` is a very common bug.
- **`merge()` returns a NEW managed object** — The original detached object remains detached. This is a famous interview trap.

---

### 2. Lazy vs Eager Loading & N+1 Problem

#### ❓ Interview Question
> *"What is the N+1 problem? How do you detect and fix it in a Spring Boot application?"*

#### 📖 Deep Explanation

**The N+1 Problem — The #1 Hibernate Performance Bug**

```java
// Entity mapping
@Entity
public class Customer {
    @Id
    private Long id;
    private String name;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders;
}

// Service — looks innocent, but...
@Transactional(readOnly = true)
public List<CustomerDTO> getAllCustomers() {
    List<Customer> customers = customerRepository.findAll(); // 1 query: SELECT all customers

    return customers.stream()
        .map(c -> {
            int orderCount = c.getOrders().size(); // N queries: 1 per customer!
            return new CustomerDTO(c.getName(), orderCount);
        })
        .collect(Collectors.toList());
}
// If there are 100 customers: 1 + 100 = 101 SQL queries!
// This is the N+1 problem.
```

**Fix #1 — JPQL Fetch Join**

```java
// Repository
@Query("SELECT DISTINCT c FROM Customer c LEFT JOIN FETCH c.orders")
List<Customer> findAllWithOrders();

// Result: 1 SQL query with a JOIN — all customers and their orders in one go
// DISTINCT prevents duplicate customers when a customer has multiple orders
```

**Fix #2 — @EntityGraph (cleaner for Spring Data)**

```java
@EntityGraph(attributePaths = {"orders"})
@Query("SELECT c FROM Customer c")
List<Customer> findAllWithOrdersGraph();
```

**Fix #3 — Batch Fetching (for large collections)**

```java
// In application.properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25

// Or per-entity
@OneToMany(mappedBy = "customer")
@BatchSize(size = 25)
private List<Order> orders;

// Hibernate sends: SELECT * FROM orders WHERE customer_id IN (?, ?, ..., ?)
// Instead of N individual queries — "N+1" becomes "N/25 + 1"
```

**Detecting N+1 in Production**

```yaml
# application.yml — enable SQL logging
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

# For production: use datasource-proxy or p6spy
# datasource-proxy counts queries per request
```

```java
// Using Hibernate Statistics
@Autowired
private EntityManagerFactory entityManagerFactory;

public void checkStats() {
    Statistics stats = entityManagerFactory
        .unwrap(SessionFactory.class)
        .getStatistics();
    stats.setStatisticsEnabled(true);

    // After your operation:
    System.out.println("Query count: " + stats.getQueryExecutionCount());
    // If count is > 1 for a single service call, investigate
}
```

**Eager vs Lazy — When to Use Each**

```java
// LAZY (default for @OneToMany, @ManyToMany) — ALWAYS prefer this
@OneToMany(fetch = FetchType.LAZY)  // Only loaded when accessed

// EAGER (default for @ManyToOne, @OneToOne) — use sparingly
@ManyToOne(fetch = FetchType.EAGER) // Loaded with every query of the parent
// Problem: EAGER loading can't be turned off per query!
// Every findById() will always join-fetch the association
```

#### ⚠️ Common Mistakes & Interview Traps
- **Using EAGER to "fix" LazyInitializationException** — This is the wrong fix. It creates performance problems everywhere the entity is loaded. The right fix is to ensure you're within a transaction, or use a fetch join.
- **LazyInitializationException outside transaction** — Accessing a lazy collection after the session closes:
  ```java
  // ❌ WRONG — transaction closed when method returns
  Customer c = customerService.findById(1L); // transaction ends here
  c.getOrders().size(); // LazyInitializationException!

  // ✅ RIGHT — access within transaction or use DTO projection
  ```

---

### 3. Caching: 1st Level, 2nd Level, Query Cache

#### ❓ Interview Question
> *"Explain Hibernate's caching layers. When would you use the second-level cache in production?"*

#### 📖 Deep Explanation

**1st Level Cache (Session Cache) — Always On**

- Scope: single `Session` / transaction
- Automatic — you can't disable it
- Why it matters: `findById` twice in the same transaction = 1 SQL query

```java
@Transactional
public void firstLevelCacheDemo() {
    // Query 1: SELECT * FROM orders WHERE id = 1
    Order o1 = orderRepo.findById(1L).orElseThrow();

    // No query! Returns same object from 1st level cache
    Order o2 = orderRepo.findById(1L).orElseThrow();

    assert o1 == o2; // Same reference
}
// Cache is cleared when transaction ends
```

**2nd Level Cache — Shared Across Sessions**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

```java
// Enable globally
// application.properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=
    org.hibernate.cache.jcache.JCacheRegionFactory

// Mark entity as cacheable
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
}
```

**Cache Concurrency Strategies**

| Strategy | Use Case |
|----------|----------|
| `READ_ONLY` | Reference data that never changes (countries, currencies) |
| `READ_WRITE` | Entities updated occasionally (product catalog) |
| `NONSTRICT_READ_WRITE` | Tolerates brief stale reads (OK for low-consistency scenarios) |
| `TRANSACTIONAL` | Full transaction support (requires JTA) |

**Query Cache — Cache the Result Set**

```java
// Enable
spring.jpa.properties.hibernate.cache.use_query_cache=true

// Usage
@Query("SELECT p FROM Product p WHERE p.category = :cat")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Product> findByCategory(@Param("cat") String category);

// Or with EntityManager
List<Product> result = em.createQuery("SELECT p FROM Product p WHERE p.active = true", Product.class)
    .setHint("org.hibernate.cacheable", true)
    .getResultList();
```

#### 🏭 Real-World Scenario
*A product catalog API served 10K requests/second. The `/products` endpoint queried 500 products that changed once per day. Without caching: 10K DB queries/second. With 2nd-level cache + query cache: ~10 DB queries/day. The cache paid for itself in 5 minutes.*

#### ⚠️ Common Mistakes & Interview Traps
- **Caching entities that change frequently** — The cache becomes a source of stale data. 2nd level cache is for relatively stable data.
- **Query cache without 2nd level cache** — Query cache stores entity IDs. On cache hit, it still needs to load each entity. Without 2nd level cache, it issues N queries to load them. You need BOTH.
- **Not configuring cache eviction** — Without TTL/max-size, the cache grows unbounded and can OOM the JVM.

---

### 4. Transactions & Propagation

#### ❓ Interview Question
> *"What does `@Transactional(propagation = Propagation.REQUIRES_NEW)` do? Give me a real use case where it matters."*

#### 📖 Deep Explanation

**Transaction Propagation Types**

```java
// REQUIRED (default): join existing transaction OR create new one
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    // Uses existing transaction if called from another @Transactional method
    // Creates new transaction if none exists
}

// REQUIRES_NEW: always suspend current transaction and create a new one
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(String event) {
    // This ALWAYS runs in its OWN transaction
    // If outer transaction rolls back, audit log is STILL saved
    auditRepo.save(new AuditEvent(event));
}

// NEVER: throw exception if transaction exists
@Transactional(propagation = Propagation.NEVER)
public void readOnlyOperation() {}

// NOT_SUPPORTED: suspend transaction, run without one
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void externalApiCall() {}

// MANDATORY: must be called within an existing transaction
@Transactional(propagation = Propagation.MANDATORY)
public void criticalOperation() {}
```

**The Classic Audit Log Use Case**

```java
@Service
public class OrderService {

    @Autowired
    private AuditService auditService;

    @Transactional
    public void processOrder(Order order) {
        try {
            validateOrder(order);
            orderRepo.save(order);
            paymentService.charge(order);

            auditService.log("ORDER_PROCESSED", order.getId()); // REQUIRES_NEW

        } catch (PaymentException e) {
            // Main transaction ROLLS BACK
            // But the audit log was in its own transaction → ALREADY COMMITTED
            // You can still see the failed attempt in the audit log!
            auditService.log("ORDER_FAILED", order.getId());
            throw e;
        }
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String event, Long orderId) {
        auditRepo.save(new AuditLog(event, orderId, Instant.now()));
        // Committed immediately, independent of outer transaction
    }
}
```

**The Self-Invocation Trap**

```java
@Service
public class OrderService {

    @Transactional
    public void outerMethod() {
        // ... some work
        this.innerMethod(); // ❌ SELF-INVOCATION — proxy is bypassed!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        // This does NOT start a new transaction!
        // Spring's @Transactional works via AOP proxy.
        // Calling 'this.innerMethod()' bypasses the proxy.
    }
}

// ✅ FIX: Inject self, or extract innerMethod into a separate bean
@Service
public class OrderService {
    @Autowired
    private OrderService self; // Spring injects the proxy

    @Transactional
    public void outerMethod() {
        self.innerMethod(); // Now calls through the proxy ✅
    }
}
```

**Rollback Rules — Know the Default Behavior**

```java
// DEFAULT: rollback on RuntimeException and Error, NOT on checked exceptions
@Transactional
public void processPayment() throws PaymentException {
    // If PaymentException is a CHECKED exception → transaction COMMITS (bug!)
    paymentGateway.charge(amount);
}

// ✅ Explicit rollback rules
@Transactional(rollbackFor = Exception.class)
public void processPayment() throws PaymentException {
    paymentGateway.charge(amount);
    // Now rolls back on ANY exception, including checked ones
}

@Transactional(noRollbackFor = ValidationException.class)
public void bulkImport(List<Record> records) {
    // Validation errors don't rollback — we handle them gracefully
}
```

#### ⚠️ Common Mistakes & Interview Traps
- **Self-invocation** — The single most common @Transactional bug. If asked "why isn't my transaction working?", self-invocation is suspect #1.
- **Checked exceptions don't rollback by default** — Shockingly many developers don't know this.
- **@Transactional on private methods** — Spring AOP proxy can't intercept private methods. No effect.

---

### 5. Mapping Strategies

#### ❓ Interview Question
> *"How would you map a @ManyToMany relationship? What are the pitfalls?"*

#### 📖 Deep Explanation

**@OneToMany — The Right Way**

```java
@Entity
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(
        mappedBy = "customer",     // 'customer' field in Order is the FK side
        cascade = CascadeType.ALL, // persist/delete orders with customer
        orphanRemoval = true,      // remove order if removed from collection
        fetch = FetchType.LAZY     // ALWAYS lazy for collections
    )
    private List<Order> orders = new ArrayList<>();

    // Helper methods to keep both sides in sync
    public void addOrder(Order order) {
        orders.add(order);
        order.setCustomer(this); // must set both sides!
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setCustomer(null);
    }
}

@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

**@ManyToMany — With Join Table**

```java
@Entity
public class Student {
    @Id
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>(); // Use Set, not List!
}

@Entity
public class Course {
    @Id
    private Long id;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

**@ManyToMany with Extra Columns — Use an Intermediate Entity**

```java
// If you need extra data in the join (e.g., enrollment date, grade)
// @ManyToMany can't store extra attributes → use an intermediate entity

@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;

    @ManyToOne
    @MapsId("studentId")
    private Student student;

    @ManyToOne
    @MapsId("courseId")
    private Course course;

    private LocalDate enrolledAt;
    private String grade;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
}
```

**Inheritance Strategies**

```java
// Strategy 1: SINGLE_TABLE (best performance, worst for constraints)
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class Payment { ... }

@Entity
@DiscriminatorValue("CREDIT")
public class CreditCardPayment extends Payment { ... }

// Strategy 2: TABLE_PER_CLASS (no joins, but no polymorphic queries)
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Notification { ... }

// Strategy 3: JOINED (cleanest model, requires joins)
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Vehicle { ... }
```

**Inheritance Strategy Comparison**

| Strategy | Performance | DB Normalization | Polymorphic Query |
|----------|-------------|-----------------|-------------------|
| SINGLE_TABLE | Best (no joins) | Poor (nullable columns) | Simple |
| TABLE_PER_CLASS | Medium | Medium | UNION (slow) |
| JOINED | Worst (joins) | Best | Join per subtype |

#### ⚠️ Common Mistakes & Interview Traps
- **Using `List` instead of `Set` for @ManyToMany** — Hibernate fires a `DELETE ALL + INSERT ALL` when you use a List and modify the collection. With `Set`, it issues precise INSERTs/DELETEs.
- **Not using helper methods for bidirectional associations** — Forgetting to set both sides leads to data inconsistency within the same session.
- **CascadeType.ALL on @ManyToMany** — Deleting one student will cascade-delete all courses! Use only `PERSIST` and `MERGE` on @ManyToMany.

---

### 6. Performance Tuning: Batching & Fetch Joins

#### ❓ Interview Question
> *"How would you bulk-insert 100,000 records efficiently using Hibernate?"*

#### 📖 Deep Explanation

**Batch Insert Configuration**

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50              # Send 50 INSERTs in one round trip
          batch_versioned_data: true   # Required for versioned entities
        order_inserts: true            # Group same-entity inserts together
        order_updates: true            # Group same-entity updates together
```

```java
@Service
@Transactional
public class BulkImportService {

    @PersistenceContext
    private EntityManager em;

    public void bulkInsert(List<Product> products) {
        int batchSize = 50;

        for (int i = 0; i < products.size(); i++) {
            em.persist(products.get(i));

            if (i % batchSize == 0 && i > 0) {
                em.flush();  // Send batch to DB
                em.clear();  // Release memory — prevent OutOfMemoryError
            }
        }
    }
}

// ⚠️ GOTCHA: GenerationType.IDENTITY disables batching in Hibernate!
// Use SEQUENCE instead:
@GeneratedValue(strategy = GenerationType.SEQUENCE,
                generator = "product_seq")
@SequenceGenerator(name = "product_seq",
                   allocationSize = 50) // Pre-allocate 50 IDs at once
private Long id;
```

**JPQL vs Native SQL for Bulk Operations**

```java
// ❌ SLOW: Load all entities, modify in memory, let dirty checking update
List<Order> orders = orderRepo.findByStatus("PENDING");
orders.forEach(o -> o.setStatus("PROCESSING")); // 1000 individual UPDATEs

// ✅ FAST: Bulk update directly in DB
@Modifying
@Transactional
@Query("UPDATE Order o SET o.status = 'PROCESSING' WHERE o.status = 'PENDING'")
int bulkUpdateStatus();
// 1 SQL statement instead of 1000

// ✅ ALSO: Clear persistence context after bulk update
// (1st level cache is now stale)
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Order o SET o.status = :newStatus WHERE o.status = :oldStatus")
int updateStatus(@Param("oldStatus") String old, @Param("newStatus") String newStatus);
```

**Projections — Only Fetch What You Need**

```java
// ❌ SLOW: Loads entire entity including all fields
List<Customer> customers = customerRepo.findAll();
// Then only uses: customer.getName(), customer.getEmail()

// ✅ FAST: Interface-based projection — only fetches needed columns
public interface CustomerSummary {
    String getName();
    String getEmail();
    Long getOrderCount();
}

@Query("SELECT c.name AS name, c.email AS email, " +
       "COUNT(o) AS orderCount " +
       "FROM Customer c LEFT JOIN c.orders o GROUP BY c.id")
List<CustomerSummary> findCustomerSummaries();

// ✅ ALSO FAST: DTO projection with constructor
@Query("SELECT new com.app.dto.CustomerDTO(c.name, c.email) FROM Customer c")
List<CustomerDTO> findCustomerDTOs();
```

---

## 🎯 Scenario-Based Questions

---

### Scenario 1: "Our API response time went from 200ms to 8 seconds overnight"

**Diagnosis Steps:**

```
1. Check: Did any data volume change significantly? (new data import, viral traffic)
2. Enable SQL logging and count queries per API call
3. Run EXPLAIN ANALYZE on the slowest queries
4. Check for missing indexes on new columns
5. Look at DB connection pool metrics — are connections being exhausted?
```

```java
// Step 1: Count queries with Spring Boot Actuator + datasource-proxy
// Add to pom.xml: net.ttddyy:datasource-proxy

// Step 2: Check connection pool (HikariCP)
// Check /actuator/metrics/hikaricp.connections.active
// If maxPoolSize = 10 and active = 10 → pool exhaustion

// Step 3: In Hibernate config:
spring.jpa.properties.hibernate.generate_statistics=true
// Then check EntityManager stats after each request
```

---

### Scenario 2: "We're seeing `LazyInitializationException` in production"

```
Root cause: Lazy collection accessed after Hibernate session closed.

Typical flow:
1. Service method (with @Transactional) loads an entity
2. Method returns — transaction + session close
3. Controller (or Jackson serializer) accesses lazy collection
4. BOOM: org.hibernate.LazyInitializationException
```

**Fix Options:**

```java
// Option 1: Use DTO projection — never expose entities to the web layer
@Transactional(readOnly = true)
public CustomerDTO getCustomer(Long id) {
    Customer c = customerRepo.findById(id).orElseThrow();
    return new CustomerDTO(c.getName(), c.getOrders().size()); // safe: within transaction
}

// Option 2: Fetch join to eagerly load what you need
@Query("SELECT c FROM Customer c LEFT JOIN FETCH c.orders WHERE c.id = :id")
Optional<Customer> findByIdWithOrders(@Param("id") Long id);

// Option 3: Open Session in View (NOT RECOMMENDED for production)
spring.jpa.open-in-view=true
// This keeps the session open for the entire HTTP request
// Dangerous: holds DB connection for duration of HTTP processing
// Disable it: spring.jpa.open-in-view=false
```

---

### Scenario 3: "We have a deadlock in our transaction logs every hour"

```
Diagnostic approach:
1. Find the deadlock in DB logs (PostgreSQL: pg_locks, MySQL: SHOW ENGINE INNODB STATUS)
2. Identify which tables/rows are involved
3. Identify which transactions are conflicting
4. Check if lock acquisition order is consistent
```

```java
// Common cause: Two service methods updating the same rows in different order
// Thread 1: updates account 1 → then account 2
// Thread 2: updates account 2 → then account 1

// Fix: Enforce consistent lock ordering
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // Sort IDs to ensure consistent lock order
    List<Long> ids = Arrays.asList(fromId, toId);
    Collections.sort(ids);

    // Lock in sorted order — prevents deadlock
    Account first = accountRepo.findByIdWithPessimisticLock(ids.get(0));
    Account second = accountRepo.findByIdWithPessimisticLock(ids.get(1));

    // Perform transfer
    if (first.getId().equals(fromId)) {
        first.debit(amount);
        second.credit(amount);
    } else {
        second.debit(amount);
        first.credit(amount);
    }
}

// Repository
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findByIdWithPessimisticLock(@Param("id") Long id);
```

---

### Scenario 4: "Our batch job processing 1M records takes 4 hours"

```java
// Problem: Processing records one by one with individual commits
// Fix: Chunked processing with batch flush + Spring Batch or manual chunking

@Transactional
public void processChunk(int offset, int limit) {
    List<Record> records = recordRepo.findSlice(PageRequest.of(offset / limit, limit));
    records.forEach(this::processRecord);
    em.flush();
    em.clear();
}

// Or use Spring Batch's ItemReader/ItemWriter with chunk-oriented processing
// @StepScope + JpaPagingItemReader handles memory management automatically
```

---

## ⚖️ Comparison & Decision Making

---

### Native SQL vs JPQL vs Criteria API

| Factor | Native SQL | JPQL | Criteria API |
|--------|-----------|------|-------------|
| **Use when** | Complex queries, DB-specific features, bulk ops | Most standard queries | Dynamic query building |
| **Type safety** | ❌ Runtime errors | ❌ Runtime errors | ✅ Compile-time |
| **DB portability** | ❌ DB-specific | ✅ DB agnostic | ✅ DB agnostic |
| **Readability** | ✅ Familiar SQL | ✅ HQL-like | ❌ Verbose |
| **Dynamic filters** | ❌ String concat (SQL injection risk) | ❌ String concat | ✅ Built for this |
| **Performance** | ✅ Full control | Good | Good |

```java
// Use Criteria API for: search endpoints with optional filters
public List<Product> searchProducts(
    String name, BigDecimal minPrice, BigDecimal maxPrice, String category) {

    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Product> cq = cb.createQuery(Product.class);
    Root<Product> root = cq.from(Product.class);
    List<Predicate> predicates = new ArrayList<>();

    if (name != null) predicates.add(cb.like(root.get("name"), "%" + name + "%"));
    if (minPrice != null) predicates.add(cb.ge(root.get("price"), minPrice));
    if (maxPrice != null) predicates.add(cb.le(root.get("price"), maxPrice));
    if (category != null) predicates.add(cb.equal(root.get("category"), category));

    cq.where(predicates.toArray(new Predicate[0]));
    return em.createQuery(cq).getResultList();
}

// Use JPQL for: standard queries with known structure
@Query("SELECT p FROM Product p WHERE p.active = true ORDER BY p.createdAt DESC")
Page<Product> findActiveProducts(Pageable pageable);

// Use Native SQL for: report queries, window functions, CTEs
@Query(value = """
    WITH ranked AS (
        SELECT *, RANK() OVER (PARTITION BY category ORDER BY price DESC) AS rk
        FROM products WHERE active = true
    )
    SELECT * FROM ranked WHERE rk <= 3
    """, nativeQuery = true)
List<Object[]> findTop3PerCategory();
```

---

### Hibernate vs Plain JDBC

| Factor | Hibernate/JPA | Plain JDBC / Spring JDBC |
|--------|--------------|--------------------------|
| **CRUD productivity** | ✅ Very fast | ❌ Boilerplate-heavy |
| **Complex queries** | ❌ Awkward | ✅ Full SQL control |
| **Bulk operations** | ❌ Slower (object overhead) | ✅ Very fast |
| **Legacy schema** | ❌ Hard to map | ✅ Flexible |
| **Learning curve** | High (many pitfalls) | Low |
| **Micro-optimizations** | ❌ Limited | ✅ Full control |

**Senior Answer**: Use Hibernate for 80% of your CRUD and domain logic. Drop to JDBC/native SQL for complex reports, bulk operations, and performance-critical paths. They're not mutually exclusive — use both in the same project.

---

### Eager vs Lazy Loading Decision Guide

```
Ask yourself: "Is this association ALWAYS needed when I load this entity?"

YES → EAGER might be OK (e.g., @ManyToOne to a parent entity — usually needed)
NO  → LAZY (and use fetch join in the specific queries that need it)

Rule of thumb:
- @ManyToOne, @OneToOne → default EAGER (can keep, but consider LAZY if problematic)
- @OneToMany, @ManyToMany → ALWAYS LAZY — never use EAGER here
```

---

## ⚡ Rapid Revision — Cheat Sheet

---

### 🗄️ SQL Cheat Sheet

```
✅ ALWAYS use parameterized queries (no string concatenation)
✅ B-Tree index: equality + range. Hash index: equality only.
✅ Leftmost prefix rule: composite index (a,b,c) serves a, a+b, a+b+c — not b or c alone
✅ Covering index: all query columns in the index → zero table reads
✅ Functions on indexed columns DISABLE the index: WHERE YEAR(date) = 2024 ❌
✅ EXPLAIN ANALYZE before any optimization decision
✅ Filter on LEFT JOIN in WHERE clause = silent INNER JOIN
✅ ROW_NUMBER = unique, RANK = gaps on ties, DENSE_RANK = no gaps
✅ READ COMMITTED is the PostgreSQL default isolation level
✅ Deadlock prevention: always acquire locks in the same order
✅ Normalization = write safety. Denormalization = read performance.
✅ Materialized views: denormalize without touching the source schema
```

### 🌿 Hibernate/JPA Cheat Sheet

```
✅ Entity states: Transient → Managed → Detached → Removed
✅ Dirty checking: no need to call save() inside a @Transactional method
✅ merge() returns a NEW managed instance — the original stays detached
✅ N+1: use JOIN FETCH, @EntityGraph, or @BatchSize to fix
✅ Never use EAGER on @OneToMany / @ManyToMany
✅ LazyInitializationException: access lazy data WITHIN the transaction
✅ Open Session in View = anti-pattern. Disable it in production.
✅ @Transactional default rollback: RuntimeException only. Add rollbackFor = Exception.class for checked exceptions.
✅ Self-invocation bypasses @Transactional proxy — #1 debugging trap
✅ REQUIRES_NEW: independent transaction — used for audit logs
✅ Use Set<> (not List<>) for @ManyToMany to avoid DELETE ALL + INSERT ALL
✅ GenerationType.IDENTITY disables batch inserts — use SEQUENCE
✅ Bulk updates with @Modifying: add clearAutomatically = true
✅ 2nd level cache: READ_ONLY for reference data, READ_WRITE for mutable
✅ Query cache needs 2nd level cache to work efficiently
✅ 1st level cache = per-session identity map. Two findById() in same tx = 1 query.
```

### 🎯 Interview Power Phrases

> *"In production, I always start with EXPLAIN ANALYZE before touching any index..."*

> *"The N+1 problem is the most common Hibernate performance issue I've seen in real codebases. My first diagnostic is to count SQL queries per API request..."*

> *"I treat @Transactional on private methods as a red flag during code review — Spring's AOP proxy can't intercept them..."*

> *"For bulk operations, I always switch to JDBC batch or native SQL — Hibernate's object overhead isn't worth it when inserting 100K+ rows..."*

> *"Open Session in View is disabled in all our services. It hides lazy loading issues and keeps DB connections open unnecessarily during HTTP processing..."*

---

### 🚀 Last-Minute Interview Reminders

1. **SQL questions** → always say "let me check the execution plan first"
2. **Hibernate questions** → always mention lazy loading and N+1 awareness
3. **Transaction questions** → mention the self-invocation trap — it shows real-world experience
4. **Performance questions** → think in layers: DB → ORM → Application → Cache
5. **Design questions** → trade-offs first, then recommendation. "It depends on..." shows seniority.
6. **Never say** "I'd add an index" without qualification. Say *"I'd check cardinality and the read/write ratio first."*
7. **Mention monitoring**: Actuator, Hibernate Statistics, slow query logs, connection pool metrics

---

*Last updated: 2025 | For use in senior-level Java/Spring Boot backend interviews*
*Technologies: Java 17+, Spring Boot 3.x, Hibernate 6.x, PostgreSQL/MySQL*
