# How to Analyze Deadlocks with SHOW ENGINE INNODB STATUS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Deadlock, InnoDB, Transaction, Diagnosis

Description: Learn how to analyze MySQL deadlocks using SHOW ENGINE INNODB STATUS to identify the conflicting transactions and the locks involved.

---

## Overview

`SHOW ENGINE INNODB STATUS` provides a detailed snapshot of InnoDB's internal state, including the most recently detected deadlock. Reading this output correctly allows you to identify which transactions conflicted, what rows were locked, and which transaction MySQL chose to roll back.

## Running the Command

```sql
SHOW ENGINE INNODB STATUS\G
```

The `\G` formats the output vertically for readability. The output contains multiple sections - look for the `LATEST DETECTED DEADLOCK` section.

## Reading the Deadlock Section

A typical deadlock report looks like this:

```text
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-06-01 14:23:45 0x7f8a1c000700
*** (1) TRANSACTION:
TRANSACTION 421938, ACTIVE 5 sec starting index read
MySQL thread id 42, OS thread handle 140234..., query id 18473 app_host user
UPDATE orders SET status = 'processing' WHERE id = 100

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 47 page no 4 n bits 72 index PRIMARY of table `mydb`.`orders`
trx id 421938 lock_mode X locks rec but not gap

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 47 page no 5 n bits 72 index PRIMARY of table `mydb`.`order_items`
trx id 421938 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:
TRANSACTION 421939, ACTIVE 3 sec starting index read
MySQL thread id 43, OS thread handle 140235..., query id 18474 app_host user
UPDATE order_items SET quantity = 2 WHERE order_id = 100

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 47 page no 5 n bits 72 index PRIMARY of table `mydb`.`order_items`
trx id 421939 lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 47 page no 4 n bits 72 index PRIMARY of table `mydb`.`orders`
trx id 421939 lock_mode X locks rec but not gap waiting

*** WE ROLL BACK TRANSACTION (2)
```

## Interpreting the Key Fields

### Transaction Details
- `TRANSACTION 421938` - the InnoDB transaction ID
- `ACTIVE 5 sec` - how long the transaction has been running
- `MySQL thread id 42` - the connection thread ID (use with `KILL` if needed)
- The SQL statement being executed when the deadlock was detected

### Lock Information
- `RECORD LOCKS` - row-level locks (as opposed to table locks)
- `space id 47 page no 4` - the InnoDB page location of the locked row
- `index PRIMARY` - the lock is on the primary key index
- `lock_mode X` - exclusive lock (X), shared lock (S), or intent lock (IX, IS)
- `locks rec but not gap` - row lock only, not a gap lock

### Lock Modes
| Mode | Meaning |
|------|---------|
| `X` | Exclusive row lock |
| `S` | Shared row lock |
| `X, GAP` | Gap lock preventing inserts in a range |
| `X, INSERT_INTENTION` | Insert intention lock |

## Identifying the Root Cause

In the example above:
- Transaction 1 locked a row in `orders` and wants a lock in `order_items`
- Transaction 2 locked a row in `order_items` and wants a lock in `orders`
- This is a classic circular wait

The fix: ensure all transactions access the same tables in the same order:

```sql
-- Both transactions should lock orders first, then order_items
BEGIN;
SELECT * FROM orders WHERE id = 100 FOR UPDATE;
SELECT * FROM order_items WHERE order_id = 100 FOR UPDATE;
-- ... make changes ...
COMMIT;
```

## Enabling Persistent Deadlock Logging

By default, only the most recent deadlock is stored. Enable logging all deadlocks to the error log:

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

Then query the error log file:

```bash
grep -A 50 "LATEST DETECTED DEADLOCK" /var/log/mysql/error.log
```

## Extracting Deadlock Information Programmatically

Use pt-deadlock-logger from Percona Toolkit to collect and store deadlock history in a table:

```bash
pt-deadlock-logger h=localhost,u=root,p=password \
  --dest h=localhost,D=deadlock_db,t=deadlocks
```

## Summary

`SHOW ENGINE INNODB STATUS` reveals the exact transactions, SQL statements, and locks involved in the most recent deadlock. Read the `HOLDS THE LOCK(S)` and `WAITING FOR THIS LOCK TO BE GRANTED` sections to understand the circular dependency. Fix deadlocks by ensuring consistent lock acquisition order across all transactions.
