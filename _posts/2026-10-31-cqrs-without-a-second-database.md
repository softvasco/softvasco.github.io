---
title: "CQRS in .NET without a second database"
kicker: "C# and CQRS / SQL Server"
description: "The read side of ledger-core is three tables in the same SQL Server database as the event store. Why the checkpoint has to sit next to those rows, what keeps the two sides apart, and what compiled EF Core queries took off the reads."
repo: ledger-core
image: /assets/cards/cqrs-without-a-second-database.png
---

A lot of CQRS diagrams show two databases: one for writes, one for reads, with something in the middle moving data between them. The read side of [ledger-core](https://github.com/softvasco/ledger-core) is three tables in the same SQL Server database as its event store. Both are built from the same connection string. Here is how that works, what still keeps the two sides apart, and when I'd split them.

## The two sides

The write side is an event store with hand-written SQL over ADO.NET. An append takes `sp_getapplock`, checks the stream version and inserts the events in one statement. The lock is there because IDENTITY hands out positions at insert time, not at commit time, so two concurrent appends could commit out of order and a reader could skip an event. The [append order post](/2026/10/event-store-append-order/) covers that in detail for PostgreSQL; on SQL Server the comment in the code says the same thing:

```csharp
// IDENTITY hands out positions at insert, not at commit, so appends go one at a time
await TakeAppendLockAsync(connection, transaction, cancellationToken).ConfigureAwait(false);
```

The read side uses EF Core ([ADR 0013](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0013-ef-core-for-the-sql-server-read-model.md)). I didn't want an ORM on the append path, where I need exact control over the lock and the insert. On the query side, statements with paging and optional date filters read better as LINQ than as SQL strings, so the two sides use different tools.

| | Write side | Read side |
|---|---|---|
| Tables | events, snapshots | account_balances, statement_lines, projection_checkpoints |
| Access | ADO.NET, hand-written SQL | EF Core, no tracking on reads |
| Shape | one row per event | one row per account, one per statement line |
| Who writes | command handlers | the projection runner only |

The statement lines table is keyed by account and stream version, and carries the balance after each line, so a statement page never has to sum anything:

```sql
create table dbo.statement_lines (
    account_id uniqueidentifier not null,
    version bigint not null,
    position bigint not null,
    kind nvarchar(20) not null,
    amount decimal(19, 4) not null,
    balance_after decimal(19, 4) not null,
    occurred_at datetimeoffset not null,
    constraint statement_lines_pk primary key (account_id, version)
);
```

The tables come from an idempotent `schema.sql`, not from EF migrations. The read model can always be rebuilt from the events, so when its shape changes the projection gets a new name and starts again from position 0.

## The checkpoint goes with the rows

A background runner polls the event store's global order every 200 ms and hands each projection the events after its checkpoint ([ADR 0012](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0012-catch-up-projections-with-checkpoints.md)). Delivery is at least once.

The first version had the runner save checkpoints through a separate store. For the in-memory read model that was fine. For tables on disk it meant two commits: the projection writes its rows, then the runner writes the checkpoint. A crash between the two replays events into rows that already have them, and the second insert of a statement line fails on the primary key.

So the SQL Server read model keeps its own checkpoint, in the same `SaveChanges` as the rows:

```csharp
await using var db = new ReadModelContext(_options);
var checkpoint = await db.Checkpoints.FindAsync([Name], cancellationToken).ConfigureAwait(false)
    ?? db.Checkpoints.Add(new CheckpointRow { Projection = Name }).Entity;

// a replay after a crash hands back events this checkpoint already covers
if (recorded.Position <= checkpoint.Position)
{
    return;
}

checkpoint.Position = recorded.Position;
if (AccountSummaryProjection.TryGetAccount(recorded.Event, out var id))
{
    await ApplyToAccountAsync(db, id, recorded, cancellationToken).ConfigureAwait(false);
}

// one SaveChanges is one transaction, so the rows and the checkpoint move together
await db.SaveChangesAsync(cancellationToken).ConfigureAwait(false);
```

The projection implements `ICheckpointedProjection`, and for those the runner reads the checkpoint from the projection and never saves one itself. The test for the crash case applies the same two events twice and checks there is still one statement line and a balance of 100.

The balance logic is `AccountSummaryProjection.Apply`, the same fold the in-memory read model uses, so the SQL Server tables and the in-memory model can't disagree about a balance.

To be fair to the two-database diagram: this part doesn't need the same database. The rule is that the checkpoint lives where the rows live. A read database on another server, with its checkpoint in it, would be just as safe, because the runner pulls from the event store and nothing has to be pushed. What the shared database saves is operational: one server to run, back up and restore, one connection string, and one container in the tests.

## Where the separation is real

Sharing a database doesn't mean sharing a model. The read side has its own tables, its own row types and its own queries, and nothing on the write side reads them. Command handlers load an account from its latest snapshot and the events after it. Queries never touch the events.

The queries are compiled with `EF.CompileAsyncQuery`. Statements use keyset paging on the stream version, so a deposit landing while someone reads page 2 can't shift page 3:

```csharp
db.StatementLines.AsNoTracking()
    .Where(l => l.AccountId == accountId && l.Version > afterVersion)
    .Where(l => from == null || l.OccurredAt >= from)
    .Where(l => to == null || l.OccurredAt < to)
    .OrderBy(l => l.Version)
    .Take(take)
```

The optional date filters are written as `from == null || ...` inside the compiled query, so there's one plan whatever filters a caller sends. I measured the three reads the API makes with BenchmarkDotNet against SQL Server in a container, one account with 1,000 deposits:

| Read | LINQ | Compiled |
|---|---:|---:|
| Balance | 0.963 ms | 0.780 ms |
| First statement page (50 lines) | 2.056 ms | 1.421 ms |
| Page for a date range late in the history | 2.283 ms | 1.631 ms |

EF Core caches translated queries anyway, but it still builds the expression tree and looks it up on every call, and a compiled query skips both. The run was noisy, so I read the gain as roughly a fifth to a third, not to the microsecond. There's no index on `occurred_at`: the primary key narrows every statement read to one account before the dates are checked, and the date range is only a little slower than the first page.

## What it costs, and when I'd split

The same database means the read model competes with appends for the same server. At this size that doesn't show in any number I have.

There are two limits I wrote down in the ADRs instead of solving. Only one runner may apply events at a time, and the lock covers one process, so two API instances on the same database would both apply the same events. And a rebuild ([ADR 0015](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0015-rebuilding-a-projection-in-place.md)) empties the tables first, so for a while `GET /accounts/{id}` can answer 404 for an account that exists. Building into new tables and swapping them at the end fixes that, and I'll do it when a rebuild is slow enough for the gap to matter.

I'd move the reads to another store when they need a different engine (full-text search over statement descriptions, say) or have to scale on their own. Neither is true yet. When it happens, the projection code stays the same and only the place its rows and checkpoint live changes.

The code is in [ledger-core](https://github.com/softvasco/ledger-core), in `src/LedgerCore.Infrastructure/ReadModels`. If you run your read model in the same database as the writes, I'm curious what made you split it later, if you did.
