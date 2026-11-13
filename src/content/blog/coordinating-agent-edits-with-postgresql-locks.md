---
title: "Coordinating Agent Edits with PostgreSQL Locks"
description: "Distributed locks are a critical part of distributed systems and are becoming increasingly important for coordinating multiple agents."
pubDate: 2026-11-12
heroImage: "../../assets/agent-postgresql-locks-hero.png"
---

Distributed locks were important even in the “old days,” before the AI era. Workers running on different machines needed to coordinate updates to the same records and avoid picking up the same task.

Today, that coordination matters even more. Multiple people can ask their agents to update the same data, alongside the services already working on it.

## Lock a ticket row inside the transaction

Suppose Alice asks her agent to close a support ticket stored in PostgreSQL while Bob asks his agent to reassign it. Our rule is that a closed ticket cannot be reassigned. Alice's agent keeps its check and change in one **transaction**, a group of database operations whose changes are saved together (committed) or undone together (rolled back).

To protect Alice's check, `SELECT ... FOR UPDATE` reads the matching rows and locks them until the transaction ends, waiting if another transaction holds a conflicting lock. Ordinary reads can still proceed, while updates, deletes, and conflicting locking reads must wait. Here we use Psycopg, a Python PostgreSQL client. [PostgreSQL row locks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)

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

The ticket change fits in a short database transaction. Now suppose Alice asks her agent to change the support email in the company Handbook (document ID `handbook-id-42`) while Bob asks his agent to rewrite the same contact paragraph. The Handbook lives in another service, and editing it may involve document and model calls. We need to coordinate that work without keeping a database transaction open.

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

For the Handbook, we choose `pg_try_advisory_lock`: it returns immediately if the lock is busy, and its session scope covers the whole edit. We also enable **autocommit**, so each database statement runs in its own transaction. A transaction-scoped lock would end with its acquisition statement in this mode; the session lock stays held during the later document and model calls. [PostgreSQL advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS) · [Psycopg autocommit](https://www.psycopg.org/psycopg3/docs/basic/transactions.html#autocommit-transactions)

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

If Alice holds the key, Bob's attempt returns `False`; he can retry later. After Alice releases it, Bob reads her saved changes before preparing his rewrite.

The `conn` connection stays open throughout the edit, so we acquire and release the lock in the same database session. If that session ends unexpectedly, PostgreSQL releases the lock even if the agent's document-editing code is still running. Losing the lock does not cancel that work. [Session lock lifetime](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)

## Other locking options

Redis and etcd are two other systems we can use for distributed locking.

### Redis

Redis is an in-memory key-value store commonly used for caching. Its ability to create a key only when that key is absent lets workers compete for the same lock key. [Redis overview](https://redis.io/docs/latest/get-started/) · [Lock design](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

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

Unlike the PostgreSQL session lock, this lock expires after 60 seconds unless extended. The example assumes the edit finishes within that window; longer edits need renewal. Expiration does not stop the callback. [Redis lock timeout](https://redis.readthedocs.io/en/stable/_modules/redis/lock.html#Lock)

Redis failover also needs care: if a replica takes over before the lock was copied to it, two agents can believe they own the lock. [Redis distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

### etcd

etcd is a distributed key-value store used for configuration and coordination across machines. Conditional transactions let workers claim a lock key only if it is absent, while a **lease** sets how long that ownership lasts unless renewed. [etcd coordination](https://etcd.io/docs/v3.6/learning/why/) · [Lease-backed lock implementation](https://docs.openstack.org/etcd3gw/latest/_modules/etcd3gw/lock.html)

Using a connected `Etcd3Client` from OpenStack's Python client, `etcd3gw`, we can wrap the same callback:

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

Here, `acquire()` tries once and returns `False` when the lock is busy. `release()` checks ownership before deleting the lock. [etcd3gw lock implementation](https://docs.openstack.org/etcd3gw/latest/_modules/etcd3gw/lock.html)

As with the Redis example, the edit must finish within 60 seconds unless we renew the lock. This client does not renew automatically; we must call `lock.refresh()` while working. Lease expiration releases the lock without stopping the callback. [Lock refresh](https://docs.openstack.org/etcd3gw/latest/api/etcd3gw.lock.html) · [Lease API](https://docs.openstack.org/etcd3gw/latest/api/etcd3gw.lease.html)

Keep the locking logic alongside the code that changes shared data: acquire before reading, keep the lock valid through the update, and release it afterward. That way, each agent using that code follows the same coordination rule.
