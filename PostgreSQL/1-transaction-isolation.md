# ISOLATION LEVELS

---

## READ COMMITED

The **partial transaction isolation** provided by Read Committed mode is adequate for many applications, and this mode
is fast and simple to use.

Read Committed in PostgreSQL **shows only committed data before a query starts**, providing a snapshot. It sees its own
transaction's updates, but **successive SELECTs may differ**.

* UPDATE, DELETE, and similar commands find rows committed at the command's start, but may encounter updates or
  deletions by other transactions, leading to waiting and re-evaluation.
* In Read Committed mode, INSERT with ON CONFLICT can insert or update rows. Unrelated transactions can unexpectedly
  affect these operations. Though unless there are unrelated errors, one of those two outcomes is guaranteed.
* MERGE allows combining INSERT, UPDATE, and DELETE actions conditionally. It re-evaluates conditions on updated rows
  and handles concurrent updates and deletions.

## REPEATABLE READ

The Repeatable Read isolation level **sees data committed before the transaction**, excluding uncommitted changes and
concurrent updates, preventing most SQL standard anomalies.

Repeatable Read shows a snapshot from the transaction's first statement, ensuring successive SELECTs see the same data,
ignoring later committed changes.

Applications using this level must be prepared to retry transactions due to serialization failures.

* UPDATE, DELETE, MERGE, SELECT FOR UPDATE, and SELECT FOR SHARE commands search for rows committed at the transaction's
  start. Concurrent updates might occur, causing a repeatable read transaction to wait. A rollback by the first
  transaction allows progression, but a commit invalidates the repeatable read, triggering a rollback.
* If an application receives this error, it should retry the transaction from the beginning, ensuring the
  previously-committed change is included.
    * Repeatable Read guarantees a stable database view, but it may not be fully consistent with serial transaction
      execution. Business rules require explicit locks to function correctly.

## SERIALIZABLE

Serializable isolation offers the strongest transaction isolation, mimicking serial execution of all committed
transactions. It monitors for inconsistencies in concurrent executions, detecting anomalies that trigger these failures,
with minimal additional blocking overhead.

* When using Serializable transactions, data read from permanent tables shouldn't be considered valid until the
  transaction commits. This applies to all transactions except deferrable read-only ones, which guarantee data validity
  upon reading. Applications must avoid relying on results from aborted transactions and instead retry until successful
  completion to prevent anomalies.
* PostgreSQL's predicate locking identifies dependencies between Serializable transactions to prevent anomalies without
  causing blocking or deadlocks, unlike other consistency methods.
* PostgreSQL uses _predicate locks_ (SIReadLock, see `pg_locks`) that depend on accessed data and can be released early
  by read-only transactions, sometimes even before completion.
* PostgreSQL's Serializable isolation can still cause unique constraint violations due to overlapping transactions, even
  with serial execution checks. Explicit key checks are needed to avoid this.

For optimal performance when relying on Serializable transactions for concurrency control, these issues should be
considered:

* Declare transactions as READ ONLY when possible.
* Control the number of active connections, using a connection pool if needed.
* Don't put more into a single transaction than needed for integrity purposes.
* Don't leave connections dangling “idle in transaction” longer than necessary. The configuration parameter
  `idle_in_transaction_session_timeout` may be used to automatically disconnect lingering sessions.
* Eliminate explicit locks, SELECT FOR UPDATE, and SELECT FOR SHARE where no longer needed due to the protections
  automatically provided by Serializable transactions.
* Insufficient memory can force predicate locks to combine, increasing serialization failures. Increasing related
  parameters can help avoid this.
* Sequential scans require relation-level locks, potentially causing serialization failures. Adjusting costs can promote
  index scans, balancing rollbacks with query time.

# PHENOMENAS REFERENCE

---

The phenomena which are prohibited at various levels are:

* **Dirty read** — a transaction reads data written by a concurrent uncommitted transaction.
* **Non-repeatable read** — a transaction re-reads data it has previously read and finds that data has been modified by
  another transaction (that committed since the initial read).
* **Phantom read** — a transaction re-executes a query returning a set of rows that satisfy a search condition and finds
  that the set of rows satisfying the condition has changed due to another recently-committed transaction.
* **Serialization anomaly** — the result of successfully committing a group of transactions is inconsistent with all
  possible orderings of running those transactions one at a time.

The SQL standard and PostgreSQL-implemented transaction isolation levels are described in the following table:

| Isolation Level      | Dirty Read             | Non-repeatable Read | Phantom Read           | Serialization Anomaly |
|----------------------|------------------------|---------------------|------------------------|-----------------------|
| **Read uncommitted** | Allowed, but not in PG | Possible            | Possible               | Possible              |
| **Read committed**   | Not possible           | Possible            | Possible               | Possible              |
| **Repeatable read**  | Not possible           | Not possible        | Allowed, but not in PG | Possible              |
| **Serializable**     | Not possible           | Not possible        | Not possible           | Not possible          |
