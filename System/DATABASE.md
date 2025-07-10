## Database
⚙️ `UNION` and `OR` in MySQL  
`UNION` processes separate result sets by executing each `SELECT` statement independently (but typically on the same thread), then merging the results. This allows MySQL to optimize each query separately and use different execution plans.

**Example scenario: Finding employees earning \>50K OR working in Sales department.**

**Using UNION:**

```sql
SELECT * FROM employees WHERE salary > 50000
UNION
SELECT * FROM employees WHERE department = 'Sales';
```

**How UNION works:**

1.  Executes first query using salary index → Result Set A
2.  Executes second query using department index → Result Set B
3.  Combines and deduplicates → Final Result

**Using OR:**

```sql
SELECT * FROM employees
WHERE salary > 50000 OR department = 'Sales';
```

Why `UNION` is often better:
    - Each subquery can use its optimal index (`salary_idx`, `department_idx`)
    - MySQL optimizes each `SELECT` independently
    - Avoids complex condition evaluation across multiple columns

**When to use OR instead of UNION:** Use OR when querying the same column with multiple values:

```sql
-- Good use of OR
SELECT * FROM employees
WHERE department = 'Sales' OR department = 'Marketing' OR department = 'IT';

-- Don't use UNION here - it's overkill and less efficient
```

**Threading note:** `UNION` typically executes on a single thread sequentially, not parallel threads. The performance benefit comes from better index usage and query optimization, not parallelization.

⚙️ Transaction in SQL following ACID  
What is a Transaction?  
A transaction is a group of SQL operations that must all succeed or all fail together. Think of it like transferring money between bank accounts, we can't have the money leave one account without arriving in the other.

**Basic Transaction Syntax in MySQL**

```sql
START TRANSACTION;

-- Our SQL operations here
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT; -- Makes changes permanent
-- OR
ROLLBACK; -- Undoes all changes if something goes wrong
```

### ACID Properties Explained
- **Atomicity - All or Nothing**
    - Either all operations in the transaction complete successfully, or none do
    - If any operation fails, the entire transaction is rolled back
    - **Example:** If the second `UPDATE` fails, the first `UPDATE` is automatically undone

- **Consistency - Data Rules Are Maintained**
    - Database remains in a valid state before and after the transaction
    - All constraints, triggers, and rules are enforced
    - **Example:** Total money in the system stays the same after the transfer

- **Isolation - Transactions Don't Interfere**
    - Concurrent transactions don't see each other's uncommitted changes
    - MySQL uses different isolation levels (READ COMMITTED, REPEATABLE READ, etc.)
    - **Example:** Another user checking balances during the transfer won't see partial results

- **Durability - Changes Are Permanent**
    - Once `COMMIT` is executed, changes survive system crashes
    - Data is written to disk, not just kept in memory
    - **Example:** Even if the server crashes after `COMMIT`, the money transfer is saved

- Key Points
    - Transactions solve the problem of partial failures in multi-step operations
    - ROLLBACK happens automatically if any error occurs (or manually with ROLLBACK command)
    - MySQL's default isolation level is REPEATABLE READ
    - Use transactions for any operation that involves multiple related changes
    - Always handle exceptions in application code to ensure proper ROLLBACK

This ensures data integrity in scenarios like financial transfers, inventory updates, or any multi-table operations.

⚙️ Race condition in DB  
A race condition occurs when the outcome of operations depends on the unpredictable timing of events. In databases, this happens when multiple transactions access shared data simultaneously, and the final result depends on which transaction "wins the race."

Classic Race Condition Example  
**Scenario:** Two people trying to buy the last item in stock
```sql
-- Both transactions run simultaneously
-- Transaction A (Person 1)
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;  -- Returns 1 (last item)
-- Context switch happens here - Transaction B runs
UPDATE products SET stock = stock - 1 WHERE id = 1;  -- Sets stock to 0
COMMIT;

-- Transaction B (Person 2)
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;  -- Also returns 1 (same item!)
UPDATE products SET stock = stock - 1 WHERE id = 1;  -- Sets stock to -1
COMMIT;
```

**Problem:** Stock becomes -1, meaning we sold 2 items when only 1 existed!  
Types of Race Conditions  

Lost Update Problem  
**What happens:** One transaction's changes get overwritten

```sql
-- Initial balance: $1000
-- Transaction A: Deposit $100
-- Transaction B: Withdraw $50
-- Both read $1000, calculate separately, last write wins
-- Result: Either $1100 or $950 (other update is lost)
```

Dirty Read Problem  
**What happens:** Reading uncommitted data that might be rolled back

```sql
-- Transaction A
UPDATE accounts SET balance = 2000 WHERE id = 1;
-- Transaction B reads balance as 2000
-- Transaction A gets ROLLBACK due to error
-- Transaction B made decisions based on invalid data
```

Non-Repeatable Read Problem  
**What happens:** Same query returns different results within one transaction

```sql
-- Transaction A
SELECT balance FROM accounts WHERE id = 1;  -- Returns $1000
-- Transaction B commits a transfer
SELECT balance FROM accounts WHERE id = 1;  -- Returns $500
-- Same transaction, different results!
```

Solutions to Race Conditions

Use Proper Isolation Levels
```sql
-- REPEATABLE READ prevents most race conditions
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;
-- Stock value remains consistent throughout transaction
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

Pessimistic Locking
```sql
-- Lock the row immediately when reading
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;
-- No other transaction can modify this row now
IF stock > 0 THEN
    UPDATE products SET stock = stock - 1 WHERE id = 1;
END IF;
COMMIT;
```

Optimistic Locking
```sql
-- Use version numbers or timestamps
START TRANSACTION;
SELECT stock, version FROM products WHERE id = 1;
-- Application logic checks stock > 0
UPDATE products 
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = @original_version;
-- If affected rows = 0, someone else modified it
COMMIT;
```

Atomic Operations
```sql
-- Single statement that checks and updates
UPDATE products 
SET stock = stock - 1
WHERE id = 1 AND stock > 0;
-- Returns affected rows count
-- If 0 rows affected, no stock available
```

Real-World Race Condition Scenarios  
E-commerce Inventory
```sql
-- WRONG: Check then act
SELECT stock FROM products WHERE id = 1;
-- Another user might buy here
UPDATE products SET stock = stock - 1 WHERE id = 1;

-- RIGHT: Atomic check and update
UPDATE products SET stock = stock - 1
WHERE id = 1 AND stock > 0;
```

Banking Transfers
```sql
-- WRONG: Separate operations
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- RIGHT: Single transaction
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1 WHERE balance >= 100;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Counter/Sequence Generation
```sql
-- WRONG: Read then increment
SELECT counter FROM sequences WHERE name = 'order_id';
UPDATE sequences SET counter = counter + 1 WHERE name = 'order_id';

-- RIGHT: Atomic increment
UPDATE sequences SET counter = counter + 1 WHERE name = 'order_id';
SELECT counter FROM sequences WHERE name = 'order_id';
```

Prevention Best Practices  
- Application Level
    - **Keep transactions short** - less time for race conditions to occur
    - **Use database constraints** - let the database enforce rules
    - **Handle retry logic** - gracefully handle failed operations due to conflicts
    - **Validate assumptions** - check if conditions still hold before acting
- Database Level
    - **Choose appropriate isolation levels** - balance consistency vs performance
    - **Use SELECT FOR UPDATE** when we plan to modify data we just read
    - **Implement proper error handling** - handle deadlocks and constraint violations
    - **Design schema with constraints** - prevent invalid states at database level

- Key Points
    - **Race conditions happen when timing matters** - multiple transactions accessing shared data
    - **ACID properties help prevent race conditions** - especially Isolation and Consistency
    - **"Check-then-act" patterns are dangerous** - state can change between check and action
    - **Atomic operations are safer** - combine check and action in single statement
    - **Higher isolation levels reduce race conditions** but hurt performance
    - **Application code must handle retries** - database might reject conflicting transactions
    - **Prevention is better than detection** - design to avoid race conditions

**Key Principle:** In concurrent systems, never assume data remains unchanged between separate operations. Always use atomic operations or proper locking mechanisms.

⚙️ One DB per service vs one DB for many service  
- **One database per service** when:
    - Services have distinct, isolated data needs.
    - We require independent scalability, technology choices (e.g., NoSQL for one, relational for another), and deployment for each service's data.
    - Data ownership is clear and strictly encapsulated within a single service.
    - This aligns with a **microservices architecture**.
    - **Industry Example:** An e-commerce platform where the `Order Service` has its own database (e.g., PostgreSQL) for order details, and the `Product Catalog Service` has its own database (e.g., MongoDB) for product information. Each service can evolve and scale its data store independently.

 - **One database for many services** when:
    - Services share a highly cohesive and interdependent data model.
    - Data consistency across services is paramount and difficult to achieve with distributed transactions.
    - Simplicity of initial setup and management is a higher priority than extreme independent scalability.
    - This is common in a **monolithic or tightly coupled service-oriented architecture**.
    - **Industry Example:** A traditional enterprise resource planning (ERP) system where `Inventory`, `Purchasing`, and `Sales` modules all share a single relational database. Changes in one module directly impact the others through shared tables and relationships within that central database.

⚙️ MySQL Isolation Levels  
Isolation levels control what a transaction can see when other transactions are running simultaneously. Think of it as setting privacy settings for the data a transaction can access from other, unfinished transactions.

### The Four Isolation Levels (Weakest to Strongest)

#### 1. READ UNCOMMITTED
**What it allows:** See uncommitted changes from other transactions.
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- Transaction 1 sees data from Transaction 2 even before Transaction 2 commits.
```
**Problem:** Dirty reads - we might get data that gets rolled back.
**Use case:** Rarely used, only for approximate counts where accuracy isn't critical.

#### 2. READ COMMITTED
**What it allows:** Only see committed changes from other transactions.
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Transaction 1 only sees data after Transaction 2 commits.
```
**Problem:** Non-repeatable reads - same query can return different results within one transaction.
**Example:** We read a balance twice in our transaction, but another transaction commits a change between our reads.

#### 3. REPEATABLE READ (MySQL Default)
**What it allows:** Same query always returns same results within a transaction.
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- If we read data at start of transaction, we'll see same data throughout.
```

**Problem:** Phantom reads - new rows might appear in range queries.
**Example:** `COUNT(*)` might change if another transaction inserts new rows.

#### 4. SERIALIZABLE (Strongest)
**What it allows:** Complete isolation - transactions run as if they're sequential.
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- No concurrency issues, but slowest performance.
```
**Problem:** Lowest performance due to heavy locking.
**Use case:** Critical operations where data consistency is more important than speed.

#### Real-World Example

**Scenario:** Two people checking and updating the same bank account.
```sql
-- Person A
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- shows $1000
-- Person B updates balance to $500 and commits
SELECT balance FROM accounts WHERE id = 1; -- what do we see?
COMMIT;
```

- Results by Isolation Level
    - **READ UNCOMMITTED:** Might see `$500` even before Person B commits.
    - **READ COMMITTED:** First read shows `$1000`, second shows `$500`.
    - **REPEATABLE READ:** Both reads show `$1000` (snapshot from start of transaction).
    - **SERIALIZABLE:** Person A waits until Person B finishes.

- Key Points
    - **SERIALIZABLE** is the safest but has low performance.
    - **MySQL default (REPEATABLE READ)** prevents most common problems.
    - **READ COMMITTED** is popular in high-concurrency applications.
    - **READ UNCOMMITTED** is rarely used due to data integrity risks.
    - Most applications stick with the default unless they have specific requirements.
    - The key tradeoff is always **consistency vs concurrency** - stronger isolation gives you cleaner data but allows fewer simultaneous operations.

⚙️ MVCC?  
MVCC creates snapshots for reads, but writes can still conflict
- Reads see consistent data from transaction start
- Writes need current data to avoid overwriting newer changes
- Problem: Read-then-write operations using stale snapshot data

**Lost Update Scenario**

Initial: Stock = 100

TxA: Reads 100, plans Stock = 90 (100-10)
TxB: Updates Stock = 95 (100-5), commits
TxA: Tries UPDATE Stock = 90 $\leftarrow$ OVERWRITES TxB's change!

**Solutions by Isolation Level**

READ COMMITTED: TxA re-reads current value (95), calculates 95-10=85 ✓
REPEATABLE READ: TxA blocks or gets current row for UPDATE ✓
SERIALIZABLE: TxA fails/retries due to phantom read detection ✓
Optimistic Locking: Version column prevents stale updates ✓

**Key Insight**

MVCC solves read consistency but **isolation levels + locking prevent lost updates** by ensuring writes use current data, not snapshot data.  

⚙️ Database locks  
Locks are mechanisms that prevent conflicts when multiple transactions try to access the same data simultaneously. Think of them like reservation systems - they ensure only authorized transactions can modify data at specific times.

Types of Locks  
- Shared Lock (S Lock) - Read Lock
    - **Purpose:** Multiple transactions can read, but none can write.
    ```sql
    SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;
    -- Multiple transactions can read this record
    -- but no one can modify it
    ```

    - **Analogy:** Like a library book: many people can read it, but no one can edit it.

- Exclusive Lock (X Lock) - Write Lock
    - **Purpose:** Only one transaction can read or write.
    ```sql
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
    -- Only this transaction can access this row
    -- All other transactions must wait
    ```

    - **Analogy:** Like editing a document: only one person can make changes at a time.

Lock Granularity (Scope)  
- Row-Level Locks
    - **What:** Locks individual rows.
    ```sql
    -- 
    UPDATE accounts SET balance = 500 WHERE id = 1;
    -- Only row with id=1 is locked
    ```

    - Advantages:
        - High concurrency - other transactions can work on different rows
        - Faster than table-level locks

- Table-Level Locks
    - **What:** Locks entire table.
    ```sql
    LOCK TABLES accounts WRITE;
    -- Entire table is locked
    -- No other transaction can access any row
    UNLOCK TABLES;
    ```
    - Advantages:
        - Simple, low overhead
    - Disadvantages:
        - Poor concurrency (Used by: MyISAM)

How Locks Work Automatically  
During SELECT (Read Operations)
```sql
-- Normal SELECT (no explicit lock)
SELECT balance FROM accounts WHERE id = 1;
-- InnoDB uses MVCC for consistent reads
```

During UPDATE/DELETE/INSERT (Write Operations)
```sql
-- UPDATE automatically gets exclusive lock
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Row is locked until transaction commits
-- Other transactions wait
```

Deadlock Example
**The Problem:** Two transactions waiting for each other.
```sql
-- Transaction A:
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
UPDATE accounts SET balance = balance + 50 WHERE id = 2;

-- Transaction B:
UPDATE accounts SET balance = balance - 30 WHERE id = 2;
UPDATE accounts SET balance = balance + 30 WHERE id = 1;
```

**MVCC's Solution:** Automatically detects deadlocks and rolls back transactions.

Lock Modes in Practice  
Optimistic Locking (Application Level)
```sql
-- Read with version
SELECT balance, version FROM accounts WHERE id = 1;

-- Update with version check
UPDATE accounts SET balance = 450, version = version + 1 
WHERE id = 1 AND version = 5;
```

Pessimistic Locking (Database Level)
```sql
-- Lock immediately
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Process business logic
UPDATE accounts SET balance = 450 WHERE id = 1;
```

Key Points  
- **Shared locks allow multiple** readers but NO writers (read-only LOCK statements)
- **Locks on biggest granularity:** row locks have fewer conflicts than table locks
- **Atomic transactions:** Every lock conflict causes deadlock
- **Deadlock resolution:** Database picks a victim to rollback and retry automatically
- **FOR UPDATE clause:** used when we need to read data we plan to modify
- **MVCC handles deadlock detection and resolution automatically**

Performance Tips
- Keep transactions short to minimize lock duration
- Access resources in consistent order across transactions
- Use appropriate isolation levels - don't use SERIALIZABLE unless necessary
- Consider optimistic locking for web applications with infrequent conflicts

The key principle: **Locks ensure data consistency but can hurt performance if held too long or acquired in conflicting patterns.**

⚙️ Clustered and non-clustered indexes
Think of MySQL's indexes as a **multi-level filing system** that lives partly on disk (as **pages**) and partly in RAM (the **buffer pool**), organized as a **B+Tree** for fast lookup.

How InnoDB Processes a Query  
When we run a query, InnoDB follows this process:  
- **Checks the buffer pool** for the tree pages it needs
- **Traverses the B+Tree** from **root → internal nodes → leaf** to find our key
- If the page isn't in RAM, **loads it from disk** into the buffer pool
- At a **clustered index** leaf, it reads the full row; at a **secondary index** leaf, it reads the key + primary-key pointer, then does a second lookup in the clustered tree to fetch the row

Key Components  
- B+Tree Structure
    - **Root Node**: Entry point, always in the buffer pool once accessed
    - **Internal Nodes**: Guide searches by key ranges
    - **Leaf Nodes**: Contain either full rows (clustered) or index entries + primary-key pointers (secondary)
- Data Pages
    - **Definition**: Fixed-size blocks (default **16 KB**) that store index records or rows
    - **Layout**: Header → data area (records) → directory & free space. Free space (~1/16th) allows in-page inserts without splitting
- Buffer Pool
    - **Purpose**: Caches pages in RAM to avoid disk I/O
    - **Behavior**: Uses an LRU algorithm split into "young" and "old" lists to track hot pages

- Clustered vs. Secondary Index
    - **Clustered Index (Primary Key)**: The table itself is stored in primary-key order. **Leaf pages = full rows**
    - **Secondary Index (Non-Clustered)**: Leaf pages store only `<indexed_columns, primary_key>` pairs. A lookup here yields the primary key, which is used to fetch the full row from the clustered tree

Step-by-Step Flow  
- **Query Issued** - Client issues `SELECT ... WHERE col = X`. InnoDB decides to use an index
- **Check Buffer Pool** - InnoDB looks for the **root page** of the B+Tree in RAM
- **Disk Fetch If Needed** - If the page isn't cached, it's read from the `.ibd` file on disk into the buffer pool
- **Traverse Internal Nodes** - Read key arrays in internal pages to pick the right branch. Repeat until a leaf node is reached
- **Leaf Lookup**:
    - **Clustered**: Binary search the leaf page for `X`, then **return the row**
    - **Secondary**: Find `<X, PK>` in the leaf, then do a **second B+Tree search** in the clustered index to fetch the row
- **Return Data** - The row is sent back to the client. If pages were dirty (modified), they'll be flushed to disk later, but we've already got our result

⚙️ Cross shard queries
![image](https://github.com/user-attachments/assets/4a481221-3f17-46de-85f2-f63b5b9b2780)

⚙️ Wide-column, Column family and Columnar DB / Column-oriented DB  
Four terms at a glance
| Label we’ll hear                     | Reality                                                                                                                              | Disk layout                                              |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------- |
| **Wide-column DB**                   | Whole database type. Rows may have thousands of sparse columns, grouped into *column families*. **Row-oriented** inside each family. | High-volume writes & key-based reads (IoT, counters).    |
| **Column family**                    | The logical sub-table that lives inside a wide-column DB. You always fetch it row-wise with the rest of the row.                     | —                                                        |
| **Columnar DB / Column-oriented DB** | Same thing: each column is stored contiguously.                                                                                      | Massive scans & aggregates (OLAP, BI, ML feature pulls). |

- Wide-column stores **are not** columnar; they merely *expose* many columns while still keeping rows together.
- Cassandra explicitly calls itself a *partitioned row store*.
- ClickHouse is a textbook column-oriented system.

**Examples**  

Wide-column (Cassandra)
```sql
-- create a “row per device, time-sorted cells” table
CREATE KEYSPACE telemetry WITH REPLICATION = {'class':'SimpleStrategy','replication_factor':1};

CREATE TABLE telemetry.device_metrics (
  device text,
  ts bigint,
  value double,
  PRIMARY KEY (device, ts)          -- row key + clustering key
) WITH CLUSTERING ORDER BY (ts DESC);

INSERT INTO telemetry.device_metrics (device, ts, value)
VALUES ('device_1', 1625097600, 45.6),
       ('device_1', 1625097660, 47.8);

SELECT ts, value FROM telemetry.device_metrics
WHERE  device = 'device_1' LIMIT 2;
```

```
 ts          | value
-------------+-------
 1625097600  |  45.6
 1625097660  |  47.8
```

```
ASCII mental picture
RowKey device_1
└── 1625097600 → 45.6
└── 1625097660 → 47.8

Disk (row-major inside SSTable):
|device_1|1625097600|45.6|1625097660|47.8|
```

*Take-away*: one hot-keyed row can contain thousands of time-series “columns” yet is still stored contiguously, perfect for ultra-fast single-row look-ups.

Columnar (ClickHouse)  
```sql
CREATE TABLE logs
(
  ts         DateTime,
  service    String,
  latency_ms UInt16
) ENGINE = MergeTree() ORDER BY ts;

INSERT INTO logs VALUES
 ('2025-07-01 10:00:00','api',122),
 ('2025-07-01 10:00:01','api',135),
 ('2025-07-01 10:00:00','auth',200);

SELECT service,
       avg(latency_ms) AS avg_latency
FROM   logs
WHERE  ts >= now() - INTERVAL 60 SECOND
GROUP  BY service;
```

```
┌─service─┬─avg_latency─┐
│ api     │       128.5 │
│ auth    │       200   │
└─────────┴─────────────┘
```

```
ASCII mental picture (column blocks on disk)

[ts]          10:00:00,10:00:01,10:00:00
[service]     "api","api","auth"
[latency_ms]  122,135,200
```

*Take-away*: only the **service** and **latency\_ms** byte-runs are read during the aggregate—huge scan speeds & compression.

When to reach for which  
| Pick this                   | If our app…                                                                                                                                                         | Sweet-spot examples                                                               |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Cassandra (wide-column)** | • Expects **>10⁵–10⁶ writes/sec**.<br>• Reads mostly by **primary key** or small time range.<br>• Can live with tunable/eventual consistency.                        | Device telemetry, message counters, user-settings, real-time leaderboards.        |
| **ClickHouse (columnar)**   | • Needs **sub-second aggregates** over **billions of rows**.<br>• Queries touch only a *few* columns per scan.<br>• Is read-heavy; data lands in batches or streams. | Product analytics, observability dashboards, financial tick analysis, log search. |

**Rule of thumb**
- Think “key-value with infinite columns” → Cassandra.
- Think “in-memory spreadsheet for petabytes” → ClickHouse.
