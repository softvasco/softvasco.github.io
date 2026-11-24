---
title: "Parking and replaying poison messages in .NET, without the dead-letter queue"
kicker: "C# and data / SQL Server and PostgreSQL"
description: "How outbox-kit parks a message the consumer can't handle in the consumer's own database, and replays it once the handler is fixed: DELETE ... OUTPUT deleted.* on SQL Server, DELETE ... RETURNING on PostgreSQL, and the inbox in the same transaction."
repo: outbox-kit
image: /assets/cards/replaying-poison-messages-from-the-database.png
---

Every consumer eventually meets a message it can't handle. Usually it's a bug in the handler, sometimes a reference to data that isn't there yet. The broker redelivers it until the delivery count runs out, and then it dead-letters it.

That is where it gets awkward. Service Bus has a dead-letter subqueue per subscription. RabbitMQ has a dead-letter exchange, if someone set one up. Kafka has nothing, and every team builds its own retry topic. Each needs different tools to look at what failed, and replaying from there usually means sending the message back to the topic, so every other subscriber gets it a second time.

This week [outbox-kit](https://github.com/softvasco/outbox-kit) got a different option: the consumer parks the message in its own database, next to the inbox table it already has, and replays it from there.

## Where the message goes

The consumer already has a database and an inbox keyed by message id and consumer name. The parked table uses the same key:

```sql
-- SQL Server (PostgreSQL: varchar, text and timestamptz)
CREATE TABLE parked_messages (
    message_id nvarchar(256) NOT NULL,
    consumer nvarchar(256) NOT NULL,
    type nvarchar(256) NOT NULL,
    payload nvarchar(max) NOT NULL,
    error nvarchar(max) NOT NULL,
    parked_at datetimeoffset NOT NULL,
    PRIMARY KEY (message_id, consumer));
```

The type, the payload as it came off the broker, and the error. Whoever fixes the bug can read that with a SELECT, which beats opening a dead-letter viewer for each broker.

## Who decides to give up

The library doesn't count attempts. The broker already does (Service Bus has `DeliveryCount`, RabbitMQ quorum queues have `x-delivery-count`), and a count kept in the database would roll back together with the handler that failed. So the consumer looks at the broker's count, and on the last try parks the message and acks it:

```csharp
var parked = new SqlServerParkedMessages(dataSource, new ParkedMessageTable());   // or PostgreSqlParkedMessages

// after the handler threw and the delivery count says this was the last try
await using var transaction = await connection.BeginTransactionAsync(ct);
await parked.ParkAsync(new ParkedMessage(message.MessageId, "billing", message.Subject, body, ex.Message, now), transaction, ct);
await transaction.CommitAsync(ct);
// then ack it
```

Once the ack goes through, the messages behind it are no longer stuck behind a message that will never work.

Parking the same message twice keeps the first row. The ack can fail after the commit, and then the broker delivers the message again; if it fails the same way, the second park must not throw. The insert is written for that on each engine. On PostgreSQL a unique violation aborts the whole transaction, so catching it isn't an option and the insert says `ON CONFLICT (message_id, consumer) DO NOTHING`. On SQL Server the existence check takes `WITH (UPDLOCK, HOLDLOCK)`, the same as the inbox insert, so two deliveries parking at once can't both pass the check and then hit the key.

## Taking a row exactly once

Replay and discard both start the same way: take the row out of the table and get its contents back, in the caller's transaction. That is one statement on each engine.

PostgreSQL:

```sql
DELETE FROM parked_messages
WHERE message_id = @message_id AND consumer = @consumer
RETURNING message_id, consumer, type, payload, error, parked_at
```

SQL Server:

```sql
DELETE FROM parked_messages
OUTPUT deleted.message_id, deleted.consumer, deleted.type,
       deleted.payload, deleted.error, deleted.parked_at
WHERE message_id = @message_id AND consumer = @consumer
```

I could have done a SELECT and then a DELETE. The single statement matters when two people press replay on the same message at the same time, which happens as soon as more than one person is on call. With one statement, the first DELETE locks the row. The second waits on that lock. When the first transaction commits, the second finds the row gone, deletes nothing and returns no rows. On SQL Server that holds under read committed snapshot too: snapshot versioning changes what plain reads see, but a DELETE still has to take the lock on the row it changes and checks it again after waiting. On PostgreSQL, read committed re-checks the row after the wait and skips it because it was deleted.

If the first transaction rolls back instead, the row is still there and the second one gets it. That is the behaviour you want: a replay that fails leaves the message parked.

There is a contract test for this that runs against both engines in Testcontainers:

```csharp
[Fact]
public async Task Two_takes_at_once_hand_the_message_to_one_of_them()
{
    await ParkAndCommit(Message("msg-1"));
    await using var firstConnection = await DataSource.OpenConnectionAsync(Token);
    await using var first = await firstConnection.BeginTransactionAsync(Token);
    Assert.NotNull(await _parked.TakeAsync("msg-1", Consumer, first, Token));

    // two people pressing replay must not run the handler twice
    var second = TakeAndCommit("msg-1");
    await first.CommitAsync(Token);

    Assert.Null(await second);
}
```

The second take starts while the first transaction still holds the row, so it can only finish after the commit, and by then the row is gone. The SQL Server test database has read committed snapshot turned on, so that case is covered too.

## Replaying through the inbox

`ReplayAsync` takes the row and runs the handler through the inbox, in the same transaction:

```csharp
var message = await parked.TakeAsync(messageId, consumer, transaction, cancellationToken);
if (message is null)
{
    return false;
}

// a redelivery may have been handled after the fix; then the inbox skips it
return await inbox.HandleOnceAsync(
    message.MessageId,
    message.Consumer,
    transaction,
    (tx, ct) => handle(message, tx, ct),
    cancellationToken);
```

So the delete, the inbox row and whatever the handler writes commit together or not at all. If the fixed handler still fails, everything rolls back and the message stays parked with its old error.

The message never goes back through the broker. Only this consumer runs it again; other subscriptions on the same topic already handled it and never see it twice.

One detail took me a moment. Parking doesn't write the message to the inbox. It would look tidy to mark it as "seen", but then the replay would find it in the inbox and skip the handler, which is the one thing the replay is for.

## The case that isn't clean

The park and the ack are two steps, and a crash between them means the broker delivers the message again. If the handler was fixed and deployed in the meantime, that delivery succeeds and the inbox records it. Now the parked row is stale. A later replay takes the row, finds the id in the inbox, skips the handler and commits, so the stale row is removed and nothing runs twice. I wrote that down in [ADR 6](https://github.com/softvasco/outbox-kit/blob/main/docs/adr/0006-park-poison-messages-in-the-consumer-database.md) rather than trying to make park and ack atomic, which they can't be with two systems.

Two more things the ADR lists. The payload is stored exactly as it came off the broker, so if the fix changes the message contract, the handler has to read the old shape for the parked ones. And nothing deletes parked rows on its own: they are there to be looked at, and a cleanup job would hide the problem.

## Same API on both engines

The C# side is shared: a base class holds the commands and the reader, and each engine passes in its three SQL statements. The differences are exactly the ones above, `ON CONFLICT DO NOTHING` against `UPDLOCK, HOLDLOCK`, `RETURNING` against `OUTPUT deleted.*`, and `LIMIT` against `TOP`.

The code is in [outbox-kit](https://github.com/softvasco/outbox-kit), under `src/OutboxKit/Inbox`. If you replay dead letters from Service Bus or RabbitMQ today, I'd like to know how you stop the other subscribers from getting them again.
