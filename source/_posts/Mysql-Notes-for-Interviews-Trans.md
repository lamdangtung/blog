---
title: Mysql Notes for Interviews - Trans
date: 2025-12-31 15:28:52
tags: [mysql, interview]
categories: [mysql]
---
> **Acknowledgement**  
> Special thanks to **shayne007** for creating the original article that this resource is based on.

---

## MySQL Basics

### Keywords
- Transaction isolation levels, Three Normal Forms (3NF)

### How to Understand the Three Normal Forms of Database Table Design
- **First Normal Form (1NF)**: 1NF is a constraint on the atomicity of attributes, requiring that **each attribute is atomic** and cannot be further divided.
- **Second Normal Form (2NF)**: 2NF is a constraint on the uniqueness of records, requiring that **each record has a unique identifier**, i.e., entity uniqueness.
- **Third Normal Form (3NF)**: 3NF is a constraint on field redundancy, meaning that no field can be derived from other fields; it requires **no redundant fields**.

### SQL Query Execution Process
- Execute the **Connector**
  - Manages connections, including authentication and authorization
- Execute query cache lookup (key-value storage of SQL statements and results)
- Execute the **Parser**
  - Lexical analysis
  - Syntax analysis
- Execute the **Optimizer**
  - Generates execution plans and selects index strategies
- Execute the **Executor**
  - Calls storage engine interfaces
  - Checks table-level permissions

---

## Database Indexes

### Keywords
- B+ Tree
- Supports range queries, reduces disk I/O, saves memory

### Why Use B+ Trees
- Compared with **skip lists**, skip lists may degrade into linked lists in extreme cases and have poor balance. Database queries require predictable query time, and skip lists consume more memory.
- Compared with **B-trees**, B-trees store data in all nodes, which is not friendly for range queries. Non-leaf nodes store data, making it difficult to fit all non-leaf nodes in memory. If non-leaf nodes cannot fit in memory, disk I/O is required even when accessing non-leaf nodes.
- Binary trees and red-black trees have too much depth, resulting in excessive disk I/O.
- **The height of a B+ tree is usually 2–4 levels (for 5–10 million records). The root node resides in memory, and querying a specific key requires at most 1–3 disk I/O operations.**
- Typically, an **auto-increment primary key** is used as the index:
  - Auto-increment primary keys are sequential, reducing **page splits** during inserts and minimizing data movement.

### Cases Where Indexes Become Ineffective
- Using `LIKE` or `!=` with fuzzy queries
- Low data selectivity (e.g., enum fields like gender)
- Special expressions, mathematical operations, and function calls
- Small data volume

### Leftmost Prefix Principle  
*(Essentially determined by the structure of composite indexes)*
- Index condition pushdown: uses data in composite indexes to check whether `WHERE` conditions are satisfied

---

## SQL Optimization

### Keywords
- Whether execution plans use indexes
- Index column selection
- Pagination query optimization

### Viewing Execution Plans
- Understanding `EXPLAIN` fields:
  - Meanings of `possible_keys`, `type`, `rows`, `extra`, etc.
  - Consider optimization when full table scans occur

### Index Column Selection
- Foreign keys
- Columns used in `WHERE`
- Columns used in `ORDER BY` to reduce sorting overhead
- Join condition columns
- Columns with high selectivity

### Optimization Strategies
- Use covering indexes to reduce table lookups
- Use `WHERE` instead of `HAVING` (filter before grouping to reduce grouping cost)
- Optimize offset in pagination queries

---

## Database Locks

### Keywords
- Types of locks, relationship between locks and indexes

### Lock Classification
- **By lock scope**
  - Row locks
  - Gap locks (left-open, right-open), work under Repeatable Read isolation level
  - Next-key locks (left-open, right-closed), work under Repeatable Read isolation level
  - Table locks
- **Optimistic locks and pessimistic locks**
- **By mutual exclusion**
  - Shared locks
  - Exclusive locks
- **Intention locks**

### Relationship Between Locks and Indexes
- In InnoDB, locks are implemented via indexes. Locking a row means locking a leaf node of the corresponding index. If no index is used, the entire table is locked.

---

## MVCC (Multi-Version Concurrency Control)

### Keywords
- Version chain, read and write operations

### Why MVCC Is Needed
- To avoid read–write blocking

### Version Chain
- Transaction ID (`trx_id`): transaction version number
- Roll pointer (`roll_ptr`)
- Undo log:
  - Version chains are stored in undo logs and resemble linked lists

### Read View
- Different Read Views see different lists of active transaction IDs (`m_ids`, uncommitted transactions)
- Relationship between Read View and isolation levels:
  - **Read Committed**: a new Read View is created for each query
  - **Repeatable Read**: a Read View is created at transaction start, and all subsequent reads use the same Read View

---

## Database Transactions

### Keywords
- ACID
- Isolation levels

### Undo Log
- Used for transaction rollback and stores version chains
- Details:
  - **Insert**: records the primary key; rollback deletes the record by primary key
  - **Delete**: records the primary key and sets a delete flag to `true`; rollback resets it to `false`
  - **Update**:
    - If the primary key is updated: delete the old record and insert a new one
    - If the primary key is not updated: record the original values of updated fields

### Redo Log
- Why redo logs are needed:
  - Sequential writes offer better performance
- Redo log buffer flushing:
  - `innodb_flush_log_at_trx_commit = 1` (default): logs are written to disk at transaction commit

### Binlog
- Binary log files
- Purposes:
  - Master–slave replication
  - Data recovery after database failures
- Flushing (`sync_binlog`):
  - `0` (default): flushing timing decided by the operating system
  - `N`: flush every N commits; smaller N leads to worse performance

### Transaction Execution Process for Data Updates
1. Read and lock the target row into the buffer pool
2. Write undo log
3. Modify data in the buffer pool
4. Write redo log
5. Commit the transaction (flushing depends on `innodb_flush_log_at_trx_commit`)
6. Flush buffer pool data to disk (not immediate)

**Sub-processes**
- If redo logs are flushed to disk but the database crashes before buffer pool data is written, MySQL replays redo logs on restart to restore data.
- If the transaction is rolled back, undo logs are used to restore data in the buffer pool and on disk.

---

## Database and Table Sharding

### Keywords
- Divide-and-conquer pattern
- Shard tables for large data volume, shard databases for high concurrency
- Sharding algorithms

### Primary Key Generation
- Auto-increment primary keys with different step sizes per database
- Snowflake algorithm

### Sharding Algorithms
- Range-based sharding (e.g., by time range)
- Hash modulo sharding
- Consistent hashing
- Lookup table method:
  - Shard mapping tables can be dynamically adjusted based on traffic
  - Mapping tables can be cached to avoid hotspots and bottlenecks

### Problems with Database and Table Sharding
- Join operation issues
- `COUNT` aggregation issues
- Transaction management issues
- Cost issues
