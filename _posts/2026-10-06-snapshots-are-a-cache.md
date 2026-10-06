---
title: "Snapshots in an event-sourced ledger are a cache"
kicker: "Event sourcing / PostgreSQL"
description: "ledger-core takes a snapshot of an account every 100 events. Treating it as a cache decided when it is taken, how it is stored, and what happens when it fails."
repo: ledger-core
image: /assets/cards/snapshots-are-a-cache.png
---

Loading an account in [ledger-core](https://github.com/softvasco/ledger-core) means reading its events and replaying them. On the benchmark laptop a 1,000-event stream reads in about 1.8 ms, so this isn't urgent, but an account that has been open for years will have far more than a thousand events and I didn't want the load time to grow with the account's age. Last week the repo got snapshots. The one rule I set before writing any of it: a snapshot is a cache. The events alone must always give the same state, and anything that goes wrong with a snapshot must look like a cache miss, never like an error.

That rule decided most of the design, so this post goes through the decisions it made for me.

## What gets stored

A snapshot is the account's state at a version:

```csharp
public sealed record AccountSnapshot(
    AccountId AccountId,
    Iban Iban,
    Currency Currency,
    AccountStatus Status,
    FreezeReason? FreezeReason,
    long Version);
```

`Version` is the number of events it covers. Loading reads the snapshot, then the events with a higher version, and replays only those. The aggregate base class has one method for this, `RestoreVersion`, and it refuses to run on an aggregate that already has events, so a snapshot can only start a fresh instance.

The account refuses to produce a snapshot while it has unsaved events:

```csharp
// a snapshot of unsaved changes could outlive a failed append and claim events that never got stored
public AccountSnapshot ToSnapshot() =>
    PendingEvents.Count == 0
        ? new AccountSnapshot(Id, Iban, Currency, Status, FreezeReason, Version)
        : throw new InvalidOperationException("Save the pending events before taking a snapshot.");
```

If the append fails and the snapshot had already been written, the next load would start from a state that includes events the store never got. The cache would be ahead of the source of truth, which is the one thing a cache must never be.

## When it is taken

The policy is one line, and the line is not the obvious one:

```csharp
// crossing a multiple, not landing on it, since one append can add several events
public bool IsDue(long versionBefore, long versionAfter) => versionAfter / Every > versionBefore / Every;
```

My first version was `versionAfter % Every == 0`. A save can append several events in one call, so a stream can go from version 98 to 103 and never land on 100. With the modulo check that account would skip its snapshot and the next chance is version 200, if that one is hit. Dividing both versions and comparing catches every crossing. The test table says it in numbers, with `Every` set to 5:

| before | after | due |
|---:|---:|:---:|
| 0 | 3 | no |
| 4 | 5 | yes |
| 5 | 6 | no |
| 3 | 7 | yes |
| 4 | 12 | yes |
| 10 | 14 | no |

The default interval is 100. Reading 99 events after a snapshot costs well under a millisecond on the benchmark numbers, and a smaller interval would just mean more snapshot writes.

## Where it lives

The snapshot goes into a second PostgreSQL table next to the events, one row per stream, with the aggregate serialized as `jsonb`:

```sql
insert into snapshots (stream_category, stream_id, version, data)
values (@category, @id, @version, @data)
on conflict (stream_category, stream_id) do update
set version = excluded.version, data = excluded.data, taken_at = now()
where snapshots.version < excluded.version
```

The `where` on the update is the part that matters. Two saves of the same account can run close together, and the slower one may carry the older state. Without the guard, the older snapshot would overwrite the newer one. The result would still be correct, because the events after version 2 get replayed either way, but the cache would get worse for no reason, and it is cheap to prevent. The contract test for it saves version 5, then version 2, and checks that version 5 is still there.

The in-memory store, used by the unit tests, has the same rule in `AddOrUpdate`, and both stores run the same abstract test class.

## When it fails

Three things can go wrong with a cache, and each one gets the cache-miss treatment.

A stored snapshot that no longer deserializes, because the snapshot shape changed in a release:

```csharp
// a snapshot from an older shape is just a cache miss, the events still rebuild the state
try
{
    return JsonSerializer.Deserialize(data, _json);
}
catch (JsonException)
{
    return null;
}
```

A `null` here means "no snapshot", and the repository replays from the first event. The next save past a threshold writes a fresh snapshot in the new shape, and the stream heals on its own. Events need upcasters when their shape changes, because they are forever. Snapshots don't, because they can be thrown away.

A snapshot write that fails after the events were stored:

```csharp
// the events are stored by now, so a failed snapshot must not look like a failed save
private async Task TrySnapshotAsync(StreamId stream, Account account, CancellationToken cancellationToken)
{
    try
    {
        await snapshots.SaveAsync(stream, account.Version, account.ToSnapshot(), cancellationToken);
    }
    catch (Exception e) when (e is not OperationCanceledException)
    {
        LogSnapshotFailed(logger, e, stream, account.Version);
    }
}
```

By the time this runs, the append has committed. If the snapshot threw, the caller would see the save as failed and retry a command that has already succeeded, and the account would get a second deposit. So it logs a warning and moves on. Cancellation is the exception to the exception: if the caller cancelled, it should hear about it.

And a snapshot store that is down entirely: `LoadAsync` on the Postgres store will throw in that case, and I left it that way. The events table is in the same database, so if one is down the other is too. A separate cache server would be a different conversation.

## The test that pins the rule

The one test I'd keep if I had to delete the others loads an account with snapshots on, wipes the snapshot store, loads it again, and compares:

```csharp
[Fact]
public async Task Loading_with_or_without_snapshots_gives_the_same_account()
{
    var repository = Repository(every: 2);
    var account = await OpenAndFlip(repository, flips: 5);

    var withSnapshots = await repository.LoadAsync(account.Id, Token);
    _snapshots.Clear();
    var fromEventsOnly = await repository.LoadAsync(account.Id, Token);

    Assert.Equal(fromEventsOnly!.ToSnapshot(), withSnapshots!.ToSnapshot());
    Assert.Equal(6, withSnapshots.Version);
}
```

With an interval of 2 and six events, the loaded account went through snapshots at versions 2, 4 and 6. If `FromSnapshot` restored a field wrongly, or `Apply` and the snapshot disagreed on what a freeze does to the state, this test fails. Everything else about snapshots is an optimisation; this test is the correctness.

I haven't benchmarked the snapshot path itself yet. The current numbers are for raw appends and stream reads, and they're in [docs/performance.md](https://github.com/softvasco/ledger-core/blob/main/docs/performance.md). The snapshot code is in [LedgerCore.Application/Snapshots](https://github.com/softvasco/ledger-core/tree/main/src/LedgerCore.Application/Snapshots) and [LedgerCore.Infrastructure/Snapshots](https://github.com/softvasco/ledger-core/tree/main/src/LedgerCore.Infrastructure/Snapshots).
