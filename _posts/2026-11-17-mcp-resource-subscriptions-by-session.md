---
title: "MCP resource subscriptions in ASP.NET Core: key them by session, not by server"
kicker: "AI and .NET APIs / Resources"
description: "Telling an MCP client that an account it attached has changed: the host says what changed, subscriptions live per session, and why keying them by the McpServer a handler receives meant unsubscribe never worked."
repo: mcp-kit
image: /assets/cards/mcp-resource-subscriptions-by-session.png
---

An MCP client can attach a resource, say an account, and show what it read. If the balance changes a minute later, the client is showing an old number. MCP has a way to fix that: the client sends `resources/subscribe` with the URI, and the server sends `notifications/resources/updated` when it changes. The client then reads it again.

I added this to [mcp-kit](https://github.com/softvasco/mcp-kit) last week. The first version passed its tests, and unsubscribe had never worked. This is how it's built now and what the bug was.

## Who knows that a resource changed

In mcp-kit a resource is a GET endpoint marked with `.AsMcpResource()`, so `/accounts/{id}` becomes `api://accounts/{id}` ([ADR 0013](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0013-resources-from-get-endpoints.md)). McpKit has no idea when an account changes. The write could come from a tool call, a normal HTTP request, a background job or another service.

I thought about guessing from the write tools: a POST to `/accounts/{id}/deposits` probably changes `/accounts/{id}`. That misses every change that doesn't go through MCP, and it fires for writes that failed half-way. So the host says what changed, after the write:

```csharp
public interface IMcpResourceNotifier
{
    ValueTask NotifyUpdatedAsync(string uri, CancellationToken cancellationToken = default);
}
```

`AddMcpKit` registers it. When the resource reads a projection, the right place to call it is where the projection is updated, not where the command ran, otherwise the client reads again before the new data is there.

A subscription is for an exact URI, and it has to match one of the resource templates. Anything else gets `ResourceNotFound`. The spec doesn't define wildcards, so there are none.

## Subscriptions need a session

McpKit runs stateless by default: no session id, every request stands alone. That's what you want behind a load balancer, but a notification has to go out later, on a connection that belongs to someone. So subscriptions only exist on a stateful server. With `Stateless = true` the server doesn't advertise `subscribe`, and `NotifyUpdatedAsync` has nobody to tell, so the host can call it either way.

The test for the basic case reads like the feature:

```csharp
await client.SubscribeToResourceAsync(Account, updates.Record, cancellationToken: Token);

var notifier = app.Services.GetRequiredService<IMcpResourceNotifier>();
await notifier.NotifyUpdatedAsync(OtherAccount, Token);
await notifier.NotifyUpdatedAsync(Account, Token);

Assert.Equal(Account, await updates.Next().WaitAsync(Wait, Token));
```

## The first version

The SDK hands the subscribe handler a request with a `Server` property of type `McpServer`, and `McpServer` has `SendNotificationAsync`. So I kept a dictionary from that server to the URIs it subscribed to:

```csharp
private readonly ConcurrentDictionary<McpServer, ConcurrentDictionary<string, byte>> _sessions = new();

public void Subscribe(McpServer session, string uri) =>
    _sessions.GetOrAdd(session, _ => new(StringComparer.Ordinal)).TryAdd(uri, 0);
```

The SDK also gave me no hook for a session ending, so when a send failed I dropped that entry. All the tests passed, including "after unsubscribing, the client hears nothing more for that URI".

## What CI found

Days later an unrelated PR failed on that unsubscribe test: expected `acc-9002`, got `acc-4471`. The test subscribes to two accounts, unsubscribes from the first, notifies both, and expects to hear only about the second. It heard about the first.

A temporary log line in `Unsubscribe` showed it: the entry was never found, and the dictionary had two keys for one session. The `McpServer` a handler gets is not the session. It's a wrapper bound to that one request (`DestinationBoundMcpServer` in SDK 2.2.0), and every request gets a new one. Subscribe stored the wrapper from the subscribe request. Unsubscribe looked up the wrapper from the unsubscribe request, found nothing and did nothing.

So why did the test ever pass? Because of my clean-up. Sending through the wrapper of a request that had already finished usually failed, the failure dropped the entry, and the client heard nothing. The test was green because the notification failed, which is the opposite of what it was checking. When the timing went the other way, the send worked and the test failed.

## Keyed by session id

The fix has two parts. Per-session state is keyed by `SessionId`, which is the same on every wrapper. And the server that sends notifications has to be the real session, which a request handler never sees. The one place the SDK hands it over is the HTTP transport's `RunSessionHandler`, which runs for the whole life of a stateful session:

```csharp
transport.RunSessionHandler = (context, server, cancellationToken) =>
    context.RequestServices.GetRequiredService<ResourceSubscriptions>().RunAsync(server, cancellationToken);
```

```csharp
public async Task RunAsync(McpServer server, CancellationToken cancellationToken)
{
    var id = server.SessionId ?? throw new InvalidOperationException("A stateful MCP session has no id.");
    _sessions[id] = new Session(server);
    try
    {
        await server.RunAsync(cancellationToken);
    }
    finally
    {
        _sessions.TryRemove(id, out _);
    }
}
```

Subscribe and unsubscribe now look the session up by the id on the request's wrapper. The session's subscriptions go away when the session ends, so a failed send no longer has to guess that a client is gone. It's logged and the loop moves on to the next session.

`RunSessionHandler` is marked experimental in SDK 2.2.0 (`MCPEXP002`). I suppress that warning at the one line that sets it, and [ADR 0016](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0016-resource-change-notifications.md) says why. If the SDK changes it, the subscription tests are the first thing to fail.

## What's left to the host

Subscriptions live in memory on one server instance. A host with several instances needs sticky sessions, and it has to get each change to every instance, for example through the message bus it already has. McpKit doesn't do that part.

The notification carries only the URI, as the spec says. The client reads the resource again, and that read goes through the same authorization and redaction as the first one, so a notification can't leak anything the caller couldn't read.

The lesson I took from it: an object a framework passes to your handler is not necessarily the long-lived thing it looks like. If you keep state per session, key it by the session id.

The code is [ResourceSubscriptions.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/ResourceSubscriptions.cs) and the tests are in [ResourceSubscriptionsOverMcpTests.cs](https://github.com/softvasco/mcp-kit/blob/main/tests/McpKit.Tests/ResourceSubscriptionsOverMcpTests.cs).
