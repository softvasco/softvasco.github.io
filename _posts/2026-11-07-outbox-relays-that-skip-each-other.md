---
title: "Running the .NET outbox relay on every instance without sending messages twice"
kicker: "C# and messaging / PostgreSQL and SQL Server"
description: "How outbox-kit lets every instance of a service run the outbox relay: a short claim with a lease instead of a long lock, FOR UPDATE SKIP LOCKED on PostgreSQL, UPDLOCK and READPAST on SQL Server, and the tests that fail when either hint is missing."
repo: outbox-kit
image: /assets/cards/outbox-relays-that-skip-each-other.png
---

The relay in [outbox-kit](https://github.com/softvasco/outbox-kit) runs inside the service as a hosted service. A service with three instances has three relays reading the same outbox table, and until this week nothing stopped two of them from reading the same pending rows and publishing both copies. Consumers deduplicate anyway, because the outbox is at least once. But that is there for crashes. If every normal poll can produce duplicates, they stop being rare and you pay for them on the broker every time.

This post is about the change that fixed it: how a relay claims a batch, the SQL for PostgreSQL and for SQL Server, and the two tests that fail when the important part of that SQL is removed.

## The options

I looked at three ([ADR 0002](https://github.com/softvasco/outbox-kit/blob/main/docs/adr/0002-claim-batches-with-a-lease.md)):

| | How | Cost |
|---|---|---|
| One relay | leader election or a distributed lock | throughput of one instance; failover waits for the lock to expire |
| Lock for the batch | `FOR UPDATE` / `UPDLOCK` in a transaction that stays open while sending | a slow broker keeps a database transaction and its locks open for seconds |
| Claim with a lease | one short `UPDATE` sets `claimed_until`, commit, then send | two extra columns, and the lease length is a setting you have to think about |

I went with the lease. The database transaction lasts as long as one statement, and a relay that dies mid-batch doesn't need anyone to clean up after it: its rows become claimable again when `claimed_until` passes.

## The store contract

The store interface has two methods:

```csharp
public interface IOutboxStore
{
    Task<IReadOnlyList<OutboxMessage>> ClaimAsync(
        int batchSize, DateTimeOffset now, DateTimeOffset claimedUntil, CancellationToken cancellationToken);

    Task MarkDispatchedAsync(OutboxMessage message, DateTimeOffset dispatchedAt, CancellationToken cancellationToken);
}
```

`now` and `claimedUntil` come from the caller's `TimeProvider`, not from the database clock. That lets the tests move time to check that a claim runs out, without sleeping. The price is that relays compare claims using their own clocks, so the claim has to be much longer than the clock skew between instances. The default is 30 seconds, and a few seconds of skew doesn't matter against that.

## PostgreSQL: FOR UPDATE SKIP LOCKED

On PostgreSQL the claim is an `UPDATE` over a subquery that locks the rows it picks and skips the ones someone else has locked:

```sql
UPDATE outbox_messages SET claimed_until = @claimed_until
WHERE id IN (
    SELECT id FROM outbox_messages
    WHERE dispatched_at IS NULL AND (claimed_until IS NULL OR claimed_until <= @now)
    ORDER BY created_at, id
    LIMIT @batch_size
    FOR UPDATE SKIP LOCKED)
RETURNING id, type, payload, created_at, ordering_key
```

Without `FOR UPDATE SKIP LOCKED`, two relays running this at the same moment both read the same ids in the subquery. The second `UPDATE` then waits on the first one's row locks. Once those are released it goes ahead, because the outer `UPDATE` only checks that the id is in the list, and it still is. Both relays get the same messages back.

## SQL Server: UPDLOCK, READPAST, ROWLOCK

SQL Server has no `SKIP LOCKED`, but table hints do the same job. The claim is an `UPDATE` through a `TOP` CTE:

```sql
WITH batch AS (
    SELECT TOP (@batch_size) id, type, payload, created_at, ordering_key, claimed_until
    FROM outbox_messages WITH (UPDLOCK, READPAST, ROWLOCK)
    WHERE dispatched_at IS NULL AND (claimed_until IS NULL OR claimed_until <= @now)
    ORDER BY created_at, id)
UPDATE batch SET claimed_until = @claimed_until
OUTPUT inserted.id, inserted.type, inserted.payload, inserted.created_at, inserted.ordering_key
```

`READPAST` skips rows another transaction has locked. `UPDLOCK` takes an update lock on the rows it reads, so a second relay's `READPAST` has something to skip. `ROWLOCK` keeps the locks on rows, so one relay doesn't lock a whole page of rows the others could have claimed.

`UPDLOCK` matters more on SQL Server than it looks. Most teams run with `READ_COMMITTED_SNAPSHOT` on, and in that mode a plain read takes no shared locks and returns the last committed version of the row. `UPDLOCK` makes the CTE read the current rows and lock them. The SQL Server tests create their database with snapshot on for that reason:

```csharp
// most teams on SQL Server turn read committed snapshot on, and READPAST has to work there too
await using var create = new SqlCommand(
    $"CREATE DATABASE {Database}; ALTER DATABASE {Database} SET READ_COMMITTED_SNAPSHOT ON;", master);
```

On both engines, neither `RETURNING` nor `OUTPUT` promises any order, so the store sorts the batch by `created_at` and `id` after reading it.

## The tests, and checking that they fail

One contract class runs against PostgreSQL 18 and SQL Server 2022 in Testcontainers. Two of its tests are about this change. The first puts 300 messages in the table and starts 8 relays that claim batches of 7 until nothing is left:

```csharp
var relays = Enumerable.Range(0, 8).Select(_ => Task.Run(async () =>
{
    IReadOnlyList<OutboxMessage> batch;
    while ((batch = await _store.ClaimAsync(7, Now, Now + Claim, Token)).Count > 0)
    {
        foreach (var message in batch)
        {
            claimed.Add(message.Id);
        }
    }
}));
await Task.WhenAll(relays);

Assert.Equal(Messages, claimed.Count);
Assert.Equal(Messages, claimed.Distinct().Count());
```

The second test holds a row in an open transaction from another connection and expects the claim to return the other row within 5 seconds, not wait for the lock.

A concurrency test that always passes doesn't prove much, so I removed the hints to watch the tests fail. With `FOR UPDATE SKIP LOCKED` removed from the PostgreSQL query, the 8 relays claimed 851 rows for 300 messages. That is nearly every message claimed more than once. Without `SKIP LOCKED` on PostgreSQL or `READPAST` on SQL Server, the claim waited on the held row until the 5 second timeout cancelled it, on both engines. I put the hints back and both tests pass again.

## What the dispatcher does with the claim

The claim is only useful if the dispatcher respects it. Before each send it checks the clock, and it stops once the claim has run out:

```csharp
// past the claim another relay may own the rest of the batch, and sending it here would double it
if (_timeProvider.GetUtcNow() >= claimedUntil)
{
    break;
}
```

`MarkDispatchedAsync` runs after the send, not before. A crash between the two means the message goes out twice, while marking it first would mean a crash loses it. Duplicates are the side I can live with.

## What it doesn't solve yet

- A failed send waits for its claim to run out before it is tried again. Retries with backoff and a dead-letter state are the next task, and they replace that wait.
- A message that takes longer to send than the claim can overlap with another relay. That is still within at least once.
- Two relays can claim messages with the same ordering key, so per-key order across instances isn't guaranteed yet. That has its own task too.

The code is in [outbox-kit](https://github.com/softvasco/outbox-kit), and the claim SQL lives in `src/OutboxKit/Stores`. If you've run `READPAST` queues on SQL Server under snapshot isolation, I'm curious whether you kept `UPDLOCK` or switched to `sp_getapplock` for something like this.
