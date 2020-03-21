In Concurrency Control theory, there are two ways you can deal with conflicts:

- You can avoid them, by employing a pessimistic locking mechanism (e.g. Read/Write locks, Two-Phase Locking)
- You can allow conflicts to occur, but you need to detect them using an optimistic locking mechanism (e.g. logical clock, MVCC)

## MVCC 
The main advantage of using the MVCC model of concurrency control rather than locking:
- locks acquired for querying (reading) data `do not conflict` with locks acquired for writing data
(in old-school 2PL approach readers `block` writers)
- so reading never blocks writing and writing never blocks reading.

Table- and row-level locking facilities are also available in PostgreSQL for applications which don't generally need full transaction isolation and prefer to explicitly manage particular points of conflict. However, proper use of MVCC will generally provide better performance than locks. In addition, application-defined advisory locks provide a mechanism for acquiring locks that are not tied to a single transaction.

### 4 levels of transaction isolation

Phenomena, prohibited on particular levels:
- `dirty read` - transaction reads data written by a concurrent uncommitted transaction.
- `nonrepeatable read` - transaction re-reads data it has previously read and finds that data has been modified by another transaction (that committed since the initial read).
- `phantom read` - transaction re-executes a query returning a set of rows that satisfy a search condition and finds that the set of rows satisfying the condition has changed due to another recently-committed transaction.

```
Isolation Level  | Dirty Read | Nonrep. Read | Phantom Read
-----------------------------------------------------------
Read uncommitted | +          | +            | +
Read committed   | -          | +            | +
Repeatable read  | -          | -            | + 
Serializable     | -          | -            | - 
```

### Transaction modes
```sql
SET TRANSACTION transaction_mode

where transaction_mode is one of:

    ISOLATION LEVEL { 
        SERIALIZABLE | 
        REPEATABLE READ | 
        READ COMMITTED | 
        READ UNCOMMITTED 
    }
    READ WRITE | READ ONLY
    [ NOT ] DEFERRABLE
```
###
#### `READ COMMITTED` (default) (no possible error)
A statement can only see rows committed before it began.
Concurrent situation:
If deleted row is "in progresss of update", we'll wait until it's committed, before trying to execute delete.

Good for pre-determined modified rows:
```sql
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE acctnum = 12345;
-- run from another session (UPDATE 2):  
-- UPDATE accounts SET balance = balance + 200 WHERE acctnum = 12345;
COMMIT;
```
```
-> UPDATE 2 waits to acquire lock
-> UPDATE 1 updates (balance + 100), releases lock
-> UPDATE 2 updates (balance + 100 + 200)
```

Unsuitable for commands that involve complex search conditions:
```sql
BEGIN;
UPDATE website SET hits = hits + 1;
-- run from another session:  
-- DELETE FROM website WHERE hits = 10;
COMMIT;
```
```
-> DELETE skips pre-update (9), 
   waits for UPDATE to finish to acquire lock on (10)
-> UPDATE finishes,
   changing (9) -> (10), (10) -> (11)
-> DELETE affects 0 rows (bad)
```

#### `REPEATABLE READ` (possible error)
All statements of the current transaction can only see rows committed before the first query or data-modification statement was executed in this transaction.
`ERROR:  could not serialize access due to concurrent update`.
This transaction cannot modify or lock rows changed by other transactions after the current repeatable read transaction began.

#### `SERIALIZABLE` (possible error)
All statements of the current transaction can only see rows committed before the first query or data-modification statement was executed in this transaction. If a pattern of reads and writes among concurrent serializable transactions would create a situation which could not have occurred for any serial (one-at-a-time) execution of those transactions, one of them will be rolled back with a serialization_failure SQLSTATE.

Works exactly the same as Repeatable Read except:
it checks if execution of a concurrent set of serializable transactions behave in same manner as all possible serial (one at a time) executions of those transactions.

```sql
 class | value 
-------+-------
     1 |    10
     1 |    20
     2 |   100
     2 |   200

-- Session 1
sum_class_1 = SELECT SUM(value) FROM mytab WHERE class = 1;
INSERT INTO mytab, VALUES (2, sum_class_1)

-- Session 2
sum_class_2 = SELECT SUM(value) FROM mytab WHERE class = 2;
INSERT INTO mytab, VALUES (1, sum_class_2)
```
With `Repeatable read` both would be allowed to commit.
But with `Serializable`:
`ERROR:  could not serialize access due to read/write dependencies among transactions`

## Locking

Locking is used internally by Transactions.

### Table locking
Example:
TRUNCATE (removes all rows from a set of tables) cannot safely be executed concurrently with other operations on the same table, so it obtains an exclusive lock on the table to enforce that.

### Row locking
Used when MVCC is not enough.

An exclusive row-level lock on a specific row is automatically acquired when the row is updated or deleted. 
The lock is held until the transaction commits or rolls back, just like table-level locks. Row-level locks do not affect data querying; they block only writers to the same row.

```
Locks\Actions  UPDATE	NO KEY UPDATE	SHARE*	KEY SHARE*
---------------------------------------------------------
UPDATE	       Waits	Waits	        Waits	Waits
NO KEY UPDATE  Waits	Waits           Waits   OK
SHARE	       Waits	Waits           OK      OK
KEY SHARE      Waits	OK              OK      OK
```

`UPDATE` = writing, `SHARE` = reading

`NO KEY UPDATE` - update without foreign keys
`KEY SHARE` - share with foreign keys


#### `FOR UPDATE` example
```sql
-- Session 1
INSERT INTO items VALUES ('key-1', '{"hello":"world"}');

BEGIN;
SELECT * FROM items WHERE key = 'key-1' FOR UPDATE;

-- Session 2
UPDATE items SET value = '{"hello":"globe"}' WHERE key = 'key-1';
...
< nothing happens (waiting for a lock) >

-- Session 1
UPDATE purchases SET ...;
COMMIT;
done

-- Session 2
done
```

## Transactions

Example:
```sql
BEGIN;

UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
SAVEPOINT my_before_insert_savepoint;
-- etc etc
ROLLBACK TO my_before_insert_savepoint;
COMMIT;
```

The SAVEPOINT allows to ROLLBACK to specific place.
There's automatically inserted SAVEPOINT at the beginning of each transaction, so it's rolled back in case of error.

### Wisdom from Stack Overflow

> Transactions are ACID, what's ACID?

**Atomicity**
transaction completes in an all-or-nothing manner
**Consistency** 
change to data written to the database must be valid and follow predefined rules
**Isolation** 
determines how transaction integrity is visible to other transactions (no intermediate steps visible to other transactions)
**Durability**
transactions which have been committed will be stored permanently

> Just want to know if JSON type is also comes under the transactions.

Everything is transactional and crash-safe in PostgreSQL unless explicitly documented not to be.

PostgreSQL's transactions operate on tuples, not individual fields. The data type is irrelevant. It isn't really possible to implement a data type that is not transactional in PostgreSQL.

Note that this imposes some limitations you may not immediately expect. Most importantly, because PostgreSQL relies on MVCC for concurrency control it must copy a value when that value is modified (or, sometimes, when other values in the same tuple are modified). It cannot change fields in-place. So if you have a 5MB json document in a field and you change a single integer value, the whole json document must be copied and written out with the changed value. PostgreSQL will then come along later and mark the old copy as free space that can be re-used.

### Special cases

Gaps can occur if transaction is rolled back in:
- `Sequence` objects
- `GENERATED AS IDENTITY` columns (they use Sequnce objects internally)
