---
title: "An event store on PostgreSQL: why appends take a lock"
kicker: "PostgreSQL / Event sourcing"
description: "Identity values are handed out at insert time, not at commit, so a reader that follows the global position can skip an event for good. Here is how my ledger's event store avoids that, and what it costs."
repo: ledger-core
image: /assets/cards/event-store-append-order.png
---

This week [ledger-core](https://github.com/softvasco/ledger-core) got its PostgreSQL event store. Most of it is one table and a few queries. The part that took the most thinking was a single line in the append path that takes a lock, and this post is about why it's there.

## What the store has to do

The interface has three operations:

- append events to one stream, but only if the stream is still at the version the caller loaded;
- read one stream, for rebuilding an aggregate;
- read all events from every stream in order, starting after a position, for projections.

The first two are the classic event sourcing part. The third one is where the trouble is. A projection (balances, statements) reads everything after the last position it processed, handles it, saves the new position, and repeats. That only works if, once it has seen position 8, nothing with a lower position can show up later.

## The table

```sql
create table if not exists events (
    position bigint generated always as identity primary key,
    stream_category text not null,
    stream_id uuid not null,
    version bigint not null check (version > 0),
    event_type text not null,
    data jsonb not null,
    recorded_at timestamptz not null default now(),
    constraint events_stream_version_key unique (stream_category, stream_id, version)
);
```

`version` is the event's place in its own stream, starting at 1. `position` is its place in the whole store. The unique key on (category, stream, version) means two writers can never both store version 3 of the same account.

## Positions are handed out before commit

An identity column is backed by a sequence. PostgreSQL takes the next value when the row is inserted, not when the transaction commits, and sequences ignore rollbacks.

Take two appends to different accounts running at the same time:

| Time | Transaction A | Transaction B | Reader |
|---|---|---|---|
| 1 | inserts, gets position 7 | | |
| 2 | | inserts, gets position 8 | |
| 3 | | commits | |
| 4 | | | reads after 6, gets 8, saves 8 |
| 5 | commits | | |
| 6 | | | reads after 8, gets nothing |

Position 7 is now in the table, and the projection will never read it. Nothing fails. A balance is just wrong, and the event store itself looks fine if you query it.

Gaps from rollbacks are a separate thing. An append that inserts and then rolls back burns its position, and that hole never fills. That one is harmless: the reader skips a number that will never exist. The interface says so in its comment: positions only grow, may have gaps, and readers keep the last one they saw instead of counting.

## The fix I picked

Every append takes a transaction-level advisory lock before it does anything else:

```csharp
// one append at a time, so ReadAll never sees position 8 commit before position 7
await using (var lockCommand = new NpgsqlCommand("select pg_advisory_xact_lock(@key)", connection, transaction))
{
    lockCommand.Parameters.AddWithValue("key", AppendLockKey);
    await lockCommand.ExecuteNonQueryAsync(cancellationToken).ConfigureAwait(false);
}
```

`pg_advisory_xact_lock` is held until the transaction ends, by commit or rollback, so nobody has to remember to release it. With it, transaction B in the table can't insert until A has committed. Positions are handed out in commit order, and a reader that sees 8 has already been able to see 7.

The same lock makes the version check simple. Under it, the append reads the current version of the stream and compares it with what the caller expected:

```csharp
var actualVersion = await CurrentVersionAsync(connection, transaction, stream, cancellationToken)
    .ConfigureAwait(false);
if (actualVersion != expectedVersion)
{
    throw new ConcurrencyConflictException(stream, expectedVersion, actualVersion);
}
```

In READ COMMITTED each statement gets a fresh snapshot, so this query sees whatever the previous lock holder committed. The unique key is still there as a backstop if something ever writes to the table without going through this code.

A conflict is an exception and not a `Result`, which is what the domain uses for business rule failures. Nobody broke a rule here. The command handler has to reload the account and try again, and that's a retry loop, not a message for the user.

## What it costs

All appends now go one at a time, across every stream. Two deposits to two unrelated accounts wait for each other. Each append is short (one lock, one select, one batch of inserts, commit), but it's still a single queue for the whole ledger.

I haven't measured it yet. A BenchmarkDotNet run of single appends and small batches is next on the list for this repo, and I'll put the numbers in the docs.

The other option I looked at lets appends run in parallel and makes the reader careful instead: store the transaction id on each row, and have the global reader only return rows written by transactions older than the oldest one still running (`pg_snapshot_xmin`). Writers never wait, but the read side gets more complicated, and one long-running transaction holds back every projection until it ends. For a ledger that isn't anywhere near its write limits, I'd rather keep the reader simple and change this when a benchmark tells me to.

| | Lock on append | Filter on read |
|---|---|---|
| Parallel appends | no | yes |
| Reader logic | `where position > @after` | needs `xid8` column and snapshot checks |
| Can a reader skip an event | no | no, if the filter is right |
| Moving parts | one line | a column, an index and the filter |

## Testing it

The in-memory store and the PostgreSQL store run the same abstract test class, `EventStoreContract`. The PostgreSQL run uses Testcontainers with `postgres:18-alpine`, so CI tests the real thing. One of the tests fires ten appends at the same new stream at once:

```csharp
[Fact]
public async Task Only_one_of_several_racing_appends_wins()
{
    var stream = NewStream();

    var attempts = Enumerable.Range(1, 10)
        .Select(n => Task.Run(() => _store.AppendAsync(stream, 0, Events(n), Token), Token));
    var outcome = await Task.WhenAll(attempts.Select(Succeeded));

    Assert.Single(outcome, won => won);
    Assert.Single(await ReadStream(stream));
}
```

Exactly one wins and the other nine get a `ConcurrencyConflictException`.

This test doesn't prove the read ordering from the table above, because that needs two transactions held open at the right moments. I want a test for that before the projection runner exists, probably by holding a transaction open on a second connection and checking that a ReadAll after it never skips.

The store is in [LedgerCore.Infrastructure/EventStore](https://github.com/softvasco/ledger-core/tree/main/src/LedgerCore.Infrastructure/EventStore), and the design of the ledger's money movements is in [ADR-0004](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0004-double-entry-bookkeeping-model.md).
