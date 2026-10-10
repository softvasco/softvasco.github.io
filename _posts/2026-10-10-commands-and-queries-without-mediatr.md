---
title: "Commands and queries in .NET without MediatR"
kicker: "C# / .NET 10"
description: "ledger-core sends commands and queries through its own dispatcher: about 250 lines, one handler per message, and tracing, logging and validation around every command."
repo: ledger-core
image: /assets/cards/commands-and-queries-without-mediatr.png
---

This week [ledger-core](https://github.com/softvasco/ledger-core) got its application layer: commands that change accounts (OpenAccount, Deposit, Withdraw), a query that reads them (GetAccount), and something in between that finds the right handler. In most .NET codebases that something is MediatR. Since version 13 MediatR is a commercial product with a paid licence above a revenue threshold, and older versions stay on Apache 2.0 without new work. For a repo that other people might copy, I didn't want either a licence question or a frozen dependency, so I wrote the dispatcher. The reasoning is in [ADR-0007](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0007-own-command-and-query-dispatcher.md). This post is about the code and the few decisions inside it.

## What the ledger needs

I listed what I actually use from a mediator before writing anything:

- one handler per message type, found from the DI container;
- an ordered list of steps around every command (tracing, logging, validation);
- handlers resolved from the caller's scope, so they can share a request's connection;
- a clear error when a handler is missing or registered twice.

No notifications to many handlers, no streaming, no pre and post processors. Those get added if a use case shows up. The command side, the query side, their interfaces and the registration come to 245 lines.

## The contracts

A command declares its result type, and a handler handles exactly one command:

```csharp
public interface ICommand<TResult>;

public interface ICommandHandler<in TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken cancellationToken = default);
}
```

Queries have the same shape with `IQuery<TResult>` and `IQueryHandler<TQuery, TResult>`. Callers only see `ICommandDispatcher.DispatchAsync<TResult>(ICommand<TResult>)`, so an endpoint or an MCP tool never knows which class does the work.

## One handler, and two is a bug

The dispatcher only has `ICommand<TResult>` at the call site, but the container needs the closed type `ICommandHandler<Withdraw, Result<Money>>`. That type is built with reflection once per command type and cached as a small typed invoker. After the first call, a dispatch is a dictionary lookup and a virtual call.

Inside the invoker, the first thing it does is ask for every registered handler, not one:

```csharp
// a second handler would silently take over, since the container returns the last one
var handlers = services.GetServices<ICommandHandler<TCommand, TResult>>().ToArray();
var handler = handlers.Length switch
{
    1 => handlers[0],
    0 => throw new InvalidOperationException($"No handler is registered for {typeof(TCommand).Name}."),
    _ => throw new InvalidOperationException(
        $"{handlers.Length} handlers are registered for {typeof(TCommand).Name}, expected one."),
};
```

`GetRequiredService` would have been shorter. It also returns the last registration when there are two, so a copied handler class or a test double left in a registration would replace the real one without any error. That is the bug I care most about in this layer, so both dispatchers throw and there is a test for each case.

## Behaviors around commands

Every command runs inside a list of behaviors, each one deciding whether to call the next step:

```csharp
public interface ICommandBehavior<in TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, Func<Task<TResult>> continuation, CancellationToken cancellationToken);
}
```

The dispatcher builds the pipeline by wrapping the handler call from the inside out:

```csharp
Func<Task<TResult>> pipeline = () => handler.HandleAsync(typed, cancellationToken);

// wrap from the inside out, so the first behavior registered is the first to run
foreach (var behavior in services.GetServices<ICommandBehavior<TCommand, TResult>>().Reverse())
{
    var next = pipeline;
    pipeline = () => behavior.HandleAsync(typed, next, cancellationToken);
}

return pipeline();
```

The order matters, and registration sets it: tracing, then logging, then validation, then the handler. Tracing is outermost so the span also covers a command that fails validation. Validation is innermost because it may stop the command, and I want that stop logged and traced like any other outcome.

The tracing behavior tags each span with how the command ended, and only an exception marks the span as an error:

| How it ended | `ledger.command.outcome` | Span status |
|---|---|---|
| Handler returned success | `succeeded` | unset |
| Business rule said no (insufficient funds) | `rejected`, plus `ledger.error.code` | unset |
| Validator found bad input | `invalid` | unset |
| Exception | `failed` | Error |

A withdrawal refused for insufficient funds is a normal answer from the ledger. If it set the span to Error, the error rate on a dashboard would go up every time a customer tried to overspend, and the real failures would be harder to see.

## Queries have no behaviors

The query dispatcher is the same code without the loop. Queries don't change state, so there is nothing to validate as a command, and once the HTTP API lands the request span already covers the read. If a slow query needs its own span later, a behavior list is easy to add, but I didn't want to build one for nothing.

## The read side lags, and the query says so

GetAccount doesn't load the aggregate. It reads an `AccountSummary` from a read model that a projection fills from the event store's global order. The projection is a pure fold, and it skips any event at or below the version it already has, so a reader that restarts from an older checkpoint applies nothing twice.

That makes the read model eventually consistent, and the integration test shows it directly:

```csharp
var id = (await Commands.DispatchAsync(new OpenAccount(SomeIban, Currency.Eur), Token)).Value;
await Commands.DispatchAsync(new Deposit(id, Money.Of(100m, Currency.Eur)), Token);
await Commands.DispatchAsync(new Withdraw(id, Money.Of(40.5m, Currency.Eur)), Token);

Assert.Null(await Queries.DispatchAsync(new GetAccount(id), Token));

Assert.Equal(3, await _summaries.CatchUpAsync(Token));
var summary = await Queries.DispatchAsync(new GetAccount(id), Token);
```

Right after the commands, the query returns null: the projection hasn't seen the account yet. After catching up it has applied three events and the balance is 59.50. Returning null instead of throwing means the HTTP layer can answer 404 for an unknown account and the client can retry. Whether the API should wait for the projection after a write is a question for next week's endpoints.

## What I gave up

The scan and the cached `MakeGenericMethod` happen at startup and on first use, which is fine for an API process and not fine for Native AOT. If ledger-core ever needs AOT, the registration has to move to a source generator, and at that point a source-generated mediator (martinothamar/Mediator, MIT) is the obvious thing to compare against. I also own the maintenance now, which for 245 lines with tests I'm happy to do.

The code is in [src/LedgerCore.Application](https://github.com/softvasco/ledger-core/tree/main/src/LedgerCore.Application), with the dispatcher tests next to it.
