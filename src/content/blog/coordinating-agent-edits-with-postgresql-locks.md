---
title: "Coordinating Agent Edits with PostgreSQL Locks"
description: "A support ticket and a shared document show where row locks, advisory locks, and revision checks each help agents avoid conflicting edits."
pubDate: 2026-11-12
heroImage: "../../assets/agent-postgresql-locks-hero.png"
---

Two people can ask their agents to change the same thing at once. If Alice's agent closes a support ticket while Bob's reassigns it, can Bob still make that change? If they both edit the same document, how do we avoid saving a replacement based on an old version?

The coordination depends on where the data lives. A short PostgreSQL transaction lets our code check and update a ticket row together. For a document in another service, our agents can agree on a PostgreSQL advisory lock, but the document service still needs to reject stale writes.

## Lock a ticket row inside the transaction

Suppose a closed ticket cannot be reassigned. Alice's agent checks the ticket before closing it:

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

If Alice locks the row first, Bob's update waits. After Alice commits, PostgreSQL checks Bob's `WHERE` clause against the updated row; the ticket is closed, so Bob changes nothing. If Bob updates first, Alice waits and then sees the reassigned ticket before closing it. This describes PostgreSQL's default **Read Committed** isolation level. [PostgreSQL Read Committed behavior](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)

Leaving the Psycopg connection context commits the change and releases the row lock; an exception rolls the transaction back. For this one-field rule, a conditional `UPDATE` could replace Alice's read and lock too. `SELECT FOR UPDATE` becomes useful when we need to inspect the row and make several decisions within the same transaction. [Psycopg transaction management](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)

## Coordinate a document edit with an advisory lock

Now suppose Alice asks her agent to change the support email in the company Handbook while Bob asks his agent to rewrite the same contact paragraph. The document lives in another service, and preparing either edit can take longer than a database transaction. We should not hold a ticket-style transaction open across that work.

Instead, both agents agree to acquire the same advisory lock **before reading** the Handbook. PostgreSQL tracks ownership of the lock, but it does not lock the external document: every participating editor must follow the same rule. We use `42` as the illustrative key for this one document; a real application needs a stable, distinct key for each resource. [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)

PostgreSQL offers two lifetimes for this lock:

- `pg_try_advisory_xact_lock` ends when the transaction commits or rolls back.
- `pg_try_advisory_lock` ends when explicitly unlocked or the database session ends.

Both functions return `true` if the lock is acquired and `false` immediately if another session holds it. A transaction lock is useful for short database work. In autocommit mode, its acquisition statement is its entire transaction, so it cannot cover later document calls. The session lock can cover those calls while the same database connection remains open. [PostgreSQL advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)

We pass an `edit_handbook` callback that reads the document, prepares an edit, and saves it with the revision it read:

```python
def edit_handbook_with_lock(dsn, edit_handbook):
    handbook_lock = 42
    with psycopg.connect(dsn, autocommit=True) as conn:
        acquired = conn.execute(
            "SELECT pg_try_advisory_lock(%s)", (handbook_lock,)
        ).fetchone()[0]
        if not acquired:
            return False

        try:
            edit_handbook()
            return True
        finally:
            conn.execute("SELECT pg_advisory_unlock(%s)", (handbook_lock,))
```

If Alice holds the key, Bob's attempt returns `False`; he can retry later. After Alice releases it, Bob reads her saved changes before preparing his rewrite. `autocommit=True` avoids leaving a database transaction open during document or model calls. Acquisition and release must use the same database session, so this approach is incompatible with PgBouncer transaction pooling. [Psycopg autocommit](https://www.psycopg.org/psycopg3/docs/basic/transactions.html#autocommit-transactions) · [PgBouncer pooling modes](https://www.pgbouncer.org/features.html)

## Let the document service reject stale writes

The advisory lock coordinates agents that follow our agreement. It cannot stop a human from editing the Handbook, and a lost database connection releases the lock even if Alice's process is still preparing an edit. We therefore also ask the document service to check the revision at save time. If the document has changed since Alice read it, her replacement is rejected and she must reread before trying again.

For example, the Google Docs `documents.batchUpdate` API accepts `writeControl.requiredRevisionId`. If that revision is no longer current, the update is rejected. This check protects the document even when another writer does not use our PostgreSQL lock. [Google Docs write controls](https://developers.google.com/workspace/docs/api/reference/rest/v1/documents/batchUpdate#WriteControl)

## If the shared lock service is not PostgreSQL

Redis or etcd can coordinate the same external edit. A Redis lock needs an expiry and an owner token checked atomically on release; asynchronous replica failover can lose a lock. An etcd lock is tied to a lease that must remain alive while the agent works. Either way, an expired lock does not stop an agent already running, so the document's revision check still matters. [Redis distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) · [etcd lock API](https://etcd.io/docs/v3.6/dev-guide/api_concurrency_reference_v3/)

The distinction I would keep in mind is where enforcement happens: a row lock protects a database transaction; an advisory lock is an agreement among cooperating workers; a revision check lets the document service reject an out-of-date save. For a simple ticket update, a conditional `UPDATE` may be enough. For a longer external edit, use coordination to reduce conflicts and a revision check to keep a stale edit from overwriting newer work.
