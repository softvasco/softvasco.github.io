---
title: "A RabbitMQ publisher confirm is not enough for a .NET outbox"
kicker: "C# and messaging / RabbitMQ"
description: "How the RabbitMQ transport in outbox-kit decides a send worked: publisher confirms tracked by RabbitMQ.Client 7, mandatory publishing so an unroutable message fails instead of vanishing, a confirm timeout under the relay's claim, and a fresh channel after a broker error."
repo: outbox-kit
image: /assets/cards/rabbitmq-confirms-are-not-enough.png
---

An outbox relay has one job that matters: mark a message as sent only when the broker really has it. If it marks too early, the message is gone and nobody finds out. If it marks too late, the message goes out twice, which consumers have to handle anyway because the outbox is at least once.

This week [outbox-kit](https://github.com/softvasco/outbox-kit) got a RabbitMQ transport. Most outbox examples with RabbitMQ turn on publisher confirms and stop there. That isn't enough: RabbitMQ will confirm a message that it then drops.

## What the transport has to promise

The dispatcher calls one method on `IMessageTransport`:

```csharp
public interface IMessageTransport
{
    Task SendAsync(OutboxMessage message, CancellationToken cancellationToken);
}
```

The contract is plain. If `SendAsync` returns, the dispatcher marks the row as sent. If it throws, the dispatcher records the failure and tries again later, with a delay that doubles from 1 second up to 5 minutes. After 10 failed attempts it parks the row as dead-lettered. So everything comes down to when `SendAsync` is allowed to return.

## Publisher confirms in RabbitMQ.Client 7

Without confirms, `BasicPublishAsync` returns as soon as the bytes are written to the socket. The broker may not have them yet, and a connection that drops right then loses the message after the relay has already marked it.

With confirms, the broker sends an ack once it has taken responsibility for the message. For a persistent message on a durable queue that means it has been written to disk. In RabbitMQ.Client 7 you ask for it when you open the channel, and with tracking on, `BasicPublishAsync` waits for the ack and throws on a nack:

```csharp
private static readonly CreateChannelOptions ConfirmedChannel = new(
    publisherConfirmationsEnabled: true,
    publisherConfirmationTrackingEnabled: true);
```

Before version 7 you had to call `WaitForConfirmsOrDie` yourself or keep a map of delivery tags. Now the publish call itself is the wait, which fits an outbox well: one awaited call, then mark the row.

## The ack for a message that went nowhere

The case to test publishes to a topic exchange with no queue bound for the routing key. Nothing is wrong at the protocol level. The exchange exists, it routes the message to zero queues, and the RabbitMQ docs are clear about what happens next: the broker confirms an unroutable message as soon as the exchange has found no queue for it.

So the publish returns, the relay marks the row as sent, and the message is gone. In a real system this is a binding that was renamed in a deploy, or a consumer service that declares its queue on startup and hasn't started yet. Both happen.

The fix is the `mandatory` flag. With it set, an unroutable message comes back to the publisher as a `basic.return` before the ack, and RabbitMQ.Client 7 turns that into a `PublishReturnException`:

```csharp
// mandatory, so a message no queue is bound for comes back as a failed send instead of vanishing
await channel.BasicPublishAsync(
    _options.Exchange,
    _options.RoutingKey(message),
    mandatory: true,
    Properties(message),
    Encoding.UTF8.GetBytes(message.Payload),
    timeout.Token);
```

And the test that pins it, against a real broker in Testcontainers:

```csharp
[Fact]
public async Task A_message_no_queue_is_bound_for_fails_the_send()
{
    await using var channel = await broker.Connection.CreateChannelAsync(cancellationToken: Token);
    var exchange = await DeclareTopicExchangeAsync(channel);
    await using var transport = new RabbitMqTransport(broker.Connection, new RabbitMqTransportOptions { Exchange = exchange });

    var failure = await Assert.ThrowsAsync<PublishReturnException>(() => transport.SendAsync(Message("orders.placed"), Token));

    Assert.Equal(312, failure.ReplyCode);
}
```

312 is `NO_ROUTE`. The row isn't marked as sent, and the dispatcher retries it on its schedule. If nobody binds a queue in time, the row ends up dead-lettered in the outbox table, with the reason saved next to it. That is a row someone can find and resend. A dropped message leaves nothing to find.

If a team really wants unroutable messages kept, an alternate exchange on the RabbitMQ side is the better place for that. Then the broker routes them somewhere and the confirm is honest.

## How long to wait for the ack

A confirm can take a while if the broker is under memory pressure or blocked by a disk alarm. The relay holds a claim on its batch for `ClaimDuration`, 30 seconds by default, and after that another relay may take the same rows (that part is in the [post about relays that skip each other](/2026/11/outbox-relays-that-skip-each-other/)). If a send still waiting for its ack finishes after that, the message goes out twice.

So the transport has its own timeout, linked to the caller's token:

```csharp
using var timeout = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
timeout.CancelAfter(_options.ConfirmTimeout);
```

The default is 10 seconds, and a test checks that it stays at most half of the default claim. A timeout counts as a failed send. The broker may still have the message, so a retry can duplicate it, and that is fine: the transport sets the outbox id as `MessageId`, so a consumer can deduplicate on it.

## A channel that closed under you

A RabbitMQ channel closes on some broker errors, for example publishing to an exchange that was deleted. The client doesn't reopen it. A transport that kept one channel for its whole life would fail every later send after one such error, even after the exchange was back.

So the transport checks before each publish:

```csharp
// a channel closes on a broker error (an exchange that doesn't exist, say), and the next send needs a fresh one
if (_channel is { IsOpen: true })
{
    return _channel;
}
```

and a test deletes the exchange, expects one failed send, declares it again and expects the next send to land in the queue.

## One publish at a time

Sends go through a `SemaphoreSlim(1, 1)`. Channels in RabbitMQ.Client aren't meant for concurrent publishing, and the dispatcher sends a batch in order anyway, so per-key ordering needs one at a time. It does cost throughput: each message waits for its own ack before the next one goes. Batching the confirms (publish a few, then wait for all of them) is the obvious next step if the dispatch benchmark says the transport is the bottleneck. So far the benchmark only measures the database side, and there the `UPDATE` per message is what costs the most.

## Side by side with Service Bus

The same `TransportContract` tests run against both brokers, but the reasons a send can fail differ:

| | RabbitMQ | Azure Service Bus |
|---|---|---|
| Send returns when | the broker acks (confirms on) | the service accepted the message |
| Nowhere to deliver | ack, unless published mandatory | the queue or topic must exist, or the send throws |
| Duplicates from a retry | consumer dedups on MessageId | duplicate detection on the entity, keyed on MessageId |
| Ordering by key | header, the consumer decides | SessionId |

The Service Bus side is simpler here because you send to an entity that has to exist. With RabbitMQ you send to an exchange and routing happens later, and the default for a message with no route is to drop it.

The code is in [outbox-kit](https://github.com/softvasco/outbox-kit), in `src/OutboxKit.RabbitMQ`, with the broker tests in `tests/OutboxKit.Tests/Transports`.
