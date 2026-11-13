---
title: "Coordinating Agent Edits with PostgreSQL Locks"
description: "A support ticket and a shared document show where row locks, advisory locks, and revision checks each help agents avoid conflicting edits."
pubDate: 2026-11-12
heroImage: "../../assets/agent-postgresql-locks-hero.png"
---

Distributed locks were important even in the “old days,” before the AI era. Workers running on different machines needed to coordinate updates to the same records and avoid picking up the same task.

Today, that coordination matters even more. Multiple people can ask their agents to update the same data, alongside the services already working on it.

## Lock a ticket row inside the transaction

Suppose Alice asks her agent to close a support ticket while Bob asks his agent to reassign it. Our rule is that a closed ticket cannot be reassigned. Alice's agent keeps its check and change in one **transaction**, a group of database operations whose changes are saved together (committed) or undone together (rolled back).

`SELECT ... FOR UPDATE` reads the matching rows and locks them until the transaction ends, waiting if another transaction holds a conflicting lock. Ordinary reads can still proceed, while updates, deletes, and conflicting locking reads must wait. Here we use Psycopg, a Python PostgreSQL client. [PostgreSQL row locks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)

```python
import psycopg


def close_ticket(dsn, ticket_key):
    with psycopg.connect(dsn) as conn:
        ticket = conn.execute(
            """SELECT status FROM tickets
               WHERE ticket_key = %s
               FOR UPDATE""",
            (ticket_key,),
        ).fetchone()

        if ticket is None or ticket[0] != "open":
            return False

        conn.execute(
            "UPDATE tickets SET status = 'closed' WHERE ticket_key = %s",
            (ticket_key,),
        )
    return True
```

Bob's agent uses a conditional update for reassignment:

```sql
UPDATE tickets SET assignee = 'Bob'
WHERE ticket_key = 'SUP-42' AND status = 'open';
```

If Alice locks the row first, Bob's update waits. After Alice commits, PostgreSQL checks Bob's `WHERE` clause against the updated row; the ticket is closed, so Bob changes nothing. This example uses PostgreSQL's default **Read Committed** isolation level, where each statement gets a fresh view of committed data. [PostgreSQL Read Committed behavior](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)

In Alice's code, the `with psycopg.connect(...)` block keeps her check and update in the same transaction. When the block finishes normally, Psycopg commits her change and releases the row lock. If an exception escapes the block, Psycopg rolls back the transaction and releases the lock instead. [Psycopg transaction management](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)

## Coordinate a document edit with an advisory lock

Now suppose Alice asks her agent to change the support email in the company Handbook while Bob asks his agent to rewrite the same contact paragraph. The document lives in another service, and preparing either edit can take longer than a database transaction. We should not hold a ticket-style transaction open across that work.

An **advisory lock** is a lock on an application-chosen key rather than a table row. Both agents agree to acquire the Handbook's key **before reading** it. PostgreSQL tracks ownership of that key, but it does not lock the external document: every participating editor must follow the same rule. [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)

PostgreSQL's advisory-lock functions take integer keys, so we assign the Handbook `handbook_lock_key = 42`. Each resource needs a stable, distinct key. [Advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)

PostgreSQL offers two lifetimes for this lock. A **database session** lasts for the connection to PostgreSQL and can span multiple transactions:

- `pg_try_advisory_xact_lock` ends when the transaction commits or rolls back.
- `pg_try_advisory_lock` ends when explicitly unlocked or the database session ends.

Both functions return `true` if the lock is acquired and `false` immediately if another session holds it. A transaction lock is useful for short database work. In **autocommit** mode, each statement runs in its own transaction, so a transaction lock ends with its acquisition statement. The session lock can cover later document calls while the same database connection remains open. [PostgreSQL advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS) · [Psycopg autocommit](https://www.psycopg.org/psycopg3/docs/basic/transactions.html#autocommit-transactions)

We pass an `edit_handbook` callback that reads the document, prepares an edit, and saves it with the revision it read:

```python
def edit_handbook_with_lock(dsn, edit_handbook):
    handbook_lock_key = 42
    with psycopg.connect(dsn, autocommit=True) as conn:
        acquired = conn.execute(
            "SELECT pg_try_advisory_lock(%s)", (handbook_lock_key,)
        ).fetchone()[0]
        if not acquired:
            return False

        try:
            edit_handbook()
            return True
        finally:
            conn.execute("SELECT pg_advisory_unlock(%s)", (handbook_lock_key,))
```

If Alice holds the key, Bob's attempt returns `False`; he can retry later. After Alice releases it, Bob reads her saved changes before preparing his rewrite. `autocommit=True` avoids leaving a database transaction open during document or model calls. [Psycopg autocommit](https://www.psycopg.org/psycopg3/docs/basic/transactions.html#autocommit-transactions)

Acquisition and release must use the same database session. PgBouncer's **transaction pooling** returns the server connection to a shared pool after each transaction, so later calls may use a different session. That makes it incompatible with session advisory locks. [PgBouncer pooling modes](https://www.pgbouncer.org/features.html)

## Let the document service reject stale writes

The advisory lock coordinates agents that follow our agreement. It cannot stop a human from editing the Handbook, and a lost database connection releases the lock even if Alice's process is still preparing an edit. We therefore also ask the document service to check the revision at save time. If the document has changed since Alice read it, her replacement is rejected and she must reread before trying again.

For example, the Google Docs `documents.batchUpdate` API accepts `writeControl.requiredRevisionId`. If that revision is no longer current, the update is rejected. This check protects the document even when another writer does not use our PostgreSQL lock. [Google Docs write controls](https://developers.google.com/workspace/docs/api/reference/rest/v1/documents/batchUpdate#WriteControl)

## If the shared lock service is not PostgreSQL

Redis or etcd can coordinate the same external edit. A Redis lock can use an expiring key with a unique **owner token** identifying the acquisition. Release must check that token and delete the key as one atomic operation, so an old owner cannot delete a newer lock. If a replica takes over before the lock was copied to it, two agents can believe they own the lock. [Redis distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

An etcd lock is tied to a **lease**, a period of ownership that the client keeps alive while working. Expiration releases the lock. Either way, an expired lock does not stop an agent already running, so the document's revision check still matters. [etcd lock API](https://etcd.io/docs/v3.6/dev-guide/api_concurrency_reference_v3/)

The distinction I would keep in mind is where enforcement happens: a row lock protects a database transaction; an advisory lock is an agreement among cooperating workers; a revision check lets the document service reject an out-of-date save. For a longer external edit, use coordination to reduce conflicts and a revision check to keep a stale edit from overwriting newer work.
