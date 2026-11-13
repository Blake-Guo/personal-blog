---
title: "Coordinating Agent Edits with PostgreSQL Locks"
description: "Distributed locks are a critical part of distributed systems and are becoming increasingly important for coordinating multiple agents."
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

Now suppose Alice asks her agent to change the support email in the company Handbook (document ID `handbook-id-42`) while Bob asks his agent to rewrite the same contact paragraph. The document lives in another service, and preparing either edit can take longer than a database transaction. We should not hold a ticket-style transaction open across that work.

An **advisory lock** is a lock on an application-chosen key rather than a table row. Both agents agree to acquire the Handbook's key **before reading** it. PostgreSQL tracks ownership of that key, but it does not lock the external document: every participating editor must follow the same rule. [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)

PostgreSQL's advisory-lock functions take integer keys, so we assign the Handbook `handbook_lock_key = 42`. Each resource needs a stable, distinct key. [Advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)

The functions differ in whether acquisition waits and when the lock is released. A **database session** lasts for the connection to PostgreSQL and can span multiple transactions:

| Function | If the lock is busy | Lock released when |
| --- | --- | --- |
| <code style="white-space: nowrap">pg_advisory_lock</code> | Waits | Explicitly unlocked or session ends |
| <code style="white-space: nowrap">pg_try_advisory_lock</code> | Returns `false` | Explicitly unlocked or session ends |
| <code style="white-space: nowrap">pg_advisory_xact_lock</code> | Waits | Transaction commits or rolls back |
| <code style="white-space: nowrap">pg_try_advisory_xact_lock</code> | Returns `false` | Transaction commits or rolls back |

The `try` forms return `true` on successful acquisition. [PostgreSQL advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)

In **autocommit** mode, each statement runs in its own transaction, so a transaction lock ends with its acquisition statement. The session lock can cover later document calls while the same database connection remains open. [PostgreSQL advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS) · [Psycopg autocommit](https://www.psycopg.org/psycopg3/docs/basic/transactions.html#autocommit-transactions)

We pass an `edit_handbook` callback that reads the document, prepares an edit, and saves the changes:

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

The `conn` connection stays open throughout the edit, so we acquire and release the lock in the same database session.

If the database session ends, PostgreSQL releases the lock even if the agent's document-editing code is still running. Losing the lock does not cancel that work.

## Other locking options

Redis and etcd can wrap the same Handbook edit: try to acquire its lock, run the callback only on success, then release the lock.

### Redis

With a connected `redis.Redis` client from redis-py, we can include the document ID directly in the string key `edit:handbook-id-42`. It identifies the same Handbook for which we assigned PostgreSQL's numeric key `42`:

```python
def edit_handbook_with_redis(redis_client, edit_handbook):
    lock = redis_client.lock("edit:handbook-id-42", timeout=60)
    if not lock.acquire(blocking=False):
        return False

    try:
        edit_handbook()
        return True
    finally:
        lock.release()
```

Like `pg_try_advisory_lock`, `acquire(blocking=False)` returns `False` when the lock is busy. redis-py generates an **owner token**, a unique value identifying this acquisition, and checks it atomically on release so an old owner cannot delete a newer lock. [redis-py lock API](https://redis.readthedocs.io/en/stable/lock.html) · [Release implementation](https://github.com/redis/redis-py/blob/master/redis/lock.py)

Unlike the PostgreSQL session lock, this lock expires after 60 seconds unless extended. The example assumes the edit finishes within that window; longer edits need renewal. Expiration does not stop the callback. If a replica takes over before the lock was copied to it, two agents can believe they own the lock. [Redis distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

### etcd

An etcd lock is tied to a **lease**, a period of ownership that expires unless renewed. Using a connected `Etcd3Client` from OpenStack's Python client, `etcd3gw`, we can wrap the same callback:

```python
def edit_handbook_with_etcd(etcd_client, edit_handbook):
    lock = etcd_client.lock("edit:handbook-id-42", ttl=60)
    if not lock.acquire():
        return False

    try:
        edit_handbook()
        return True
    finally:
        lock.release()
```

Here, `acquire()` tries once and returns `False` when the lock is busy. `release()` checks ownership before deleting the lock. This client does not renew the lease automatically: as with the Redis example, the edit must finish within 60 seconds, or we must call `lock.refresh()` while working. Lease expiration releases the lock without stopping the callback. [etcd3gw lock implementation](https://docs.openstack.org/etcd3gw/latest/_modules/etcd3gw/lock.html) · [Lease API](https://docs.openstack.org/etcd3gw/latest/api/etcd3gw.lease.html)

The distinction I would keep in mind is where enforcement happens: PostgreSQL makes conflicting updates wait for a row lock, while an advisory lock coordinates workers that agree to acquire the same key. Choose the lock's lifetime to cover the work those workers need to coordinate.
