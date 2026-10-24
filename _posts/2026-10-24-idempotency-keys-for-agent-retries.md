---
title: "Idempotency keys in ASP.NET Core that also cover MCP tool calls"
kicker: "AI and .NET APIs / Part 5"
description: "Part 5 of the series on making an ASP.NET Core API safe for AI agents: an Idempotency-Key that runs a POST once and replays the answer, why it had to move from a middleware to an endpoint filter, and how an MCP tool call gets the key to the endpoint."
repo: ledger-core
image: /assets/cards/idempotency-keys-for-agent-retries.png
---

An agent that calls a `deposit` tool and gets a timeout does what any client does: it calls the tool again with the same arguments. If the first call did reach the ledger, the account is now credited twice. [Part 3](/2026/10/tool-schemas-from-request-types/) built the tool schemas and Part 4 was the [error shape](/2026/10/one-error-shape-in-a-minimal-api/). This part is about retries, which agents do more than most HTTP clients.

A command is not idempotent by itself. Two deposits of 50 are two deposits, and the ledger can't tell a retry from a second payment. The client has to say which one it means, so the usual answer is to let it name the operation. Stripe does this, and there is an IETF draft for an `Idempotency-Key` header. The server runs the first request with a key and gives every repeat the stored response.

## The rules

[ledger-core](https://github.com/softvasco/ledger-core) takes the key on any POST under `/accounts`. It is optional: a POST without it runs as before. With it, the answers are:

| Request | Answer |
|---|---|
| First request with a key | runs, and the response is stored |
| Same key, same request, after it finished | the stored status and body, plus `Idempotent-Replayed: true` |
| Same key while the first is still running | 409, code `idempotency.in_progress` |
| Same key, different request | 422, code `idempotency.key_reused` |
| Empty, sent twice, or over 255 characters | 400, code `idempotency.invalid_key` |
| First request ended in a 5xx or threw | the key is released, so the retry runs for real |

Two of these took some thought. A refused command, like a withdrawal bigger than the balance, is stored and replayed too. The retry should get the answer the first request got, even if the account has changed since. And a 5xx releases the key, because nothing was promised to the client, so there is nothing to replay.

Stored responses are kept for 24 hours. A claim that never finished, because the process died, can be taken over after one minute.

## First version: a middleware

The first version was a middleware ([ADR 0009](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0009-idempotency-keys-for-post.md)). It sat in front of the endpoints, read the header, hashed the method, path and raw body, and buffered the response to store it. It worked for every HTTP test I had.

Then mcp-kit started calling the same endpoints as MCP tools. It doesn't send an HTTP request to itself. It finds the endpoint, builds a fresh `HttpContext` and calls the endpoint's `RequestDelegate` directly. A middleware is not part of that delegate. So the caller that retries the most was the one caller the middleware never saw. A tool call with a key deposited twice.

## Second version: an endpoint filter

Endpoint filters are part of the endpoint's delegate, so anyone who calls the delegate runs them. The same rules moved into a filter factory ([ADR 0010](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0010-idempotency-as-an-endpoint-filter.md)), and the group opts in:

```csharp
var accounts = routes.MapGroup("/accounts").WithTags("Accounts").WithIdempotencyKeys();
```

The factory runs once per endpoint at startup. It refuses an endpoint whose handler doesn't return an `IResult`, because then there is no response it can store. It also works out which handler arguments are the request:

```csharp
// a filter, not middleware, so callers that run the endpoint's delegate in-process get it too
public static TBuilder WithIdempotencyKeys<TBuilder>(this TBuilder builder)
    where TBuilder : IEndpointConventionBuilder =>
    builder.AddEndpointFilterFactory((factory, next) =>
    {
        if (!ReturnsResult(factory.MethodInfo))
        {
            throw new InvalidOperationException($"{factory.MethodInfo.Name} must return an IResult for its response to be stored.");
        }

        var isService = factory.ApplicationServices.GetRequiredService<IServiceProviderIsService>();
        var requestArguments = factory.MethodInfo.GetParameters()
            .Where(p => !NotPartOfTheRequest.Contains(p.ParameterType) && !isService.IsService(p.ParameterType))
            .Select(p => p.Position)
            .ToArray();

        return context => InvokeAsync(context, next, requestArguments);
    });
```

That second part changed the fingerprint. By the time a filter runs, binding has already read the body, so hashing raw bytes is no longer an option. The filter hashes the method, the path and the bound arguments serialized with the app's JSON options. Services from DI, the `CancellationToken` and the `HttpContext` are left out. Two bodies that differ only in whitespace or property order now count as the same request, which is closer to what the client meant. An in-process caller that builds its request differently from an HTTP client also gets the same hash.

The filter runs the handler's `IResult` into a buffer, stores the status, content type, `Location` and body, then copies the buffer to the real response. If the handler throws or the status is 500 or more, it releases the key instead. It stores or releases with `CancellationToken.None`, because the command may already have committed when the client goes away.

The test that would have caught the middleware gap calls the delegate the way an MCP host does:

```csharp
// an MCP host calls the endpoint's delegate directly, with no middleware in front of it
[Fact]
public async Task A_retry_that_skips_the_middleware_pipeline_still_credits_once()
{
    // ...
    var first = await InvokeInProcessAsync(deposit, account, new AmountRequest(50m, "EUR"), "deposit-4");
    var retry = await InvokeInProcessAsync(deposit, account, new AmountRequest(50m, "EUR"), "deposit-4");

    Assert.Equal(StatusCodes.Status200OK, first.Response.StatusCode);
    Assert.Equal("true", retry.Response.Headers["Idempotent-Replayed"]);
    Assert.Equal(50m, await BalanceAsync(client, account));
}
```

## Getting the key into a tool call

The filter reads a header, and a tool call is a flat JSON object of arguments. The schema from Part 3 only knew route, query and body, so a model had no way to send the key at all. I looked at three ways to fix that ([mcp-kit ADR 0003](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0003-header-arguments.md)):

| Option | Problem |
|---|---|
| mcp-kit makes up a key per tool call | each retry is a new call, so it gets a new key and the filter sees two different requests |
| Map every `[FromHeader]` parameter | the filter reads the header itself, so there is no parameter to find |
| The endpoint declares which argument becomes which header | one extra line per endpoint or group |

The first one looks the most convenient and is useless: the whole point is that the retry carries the same key as the first attempt, and only the caller knows it is retrying. So the endpoint declares it:

```csharp
.WithMcpToolHeader("Idempotency-Key", "idempotency_key", "reuse it on a retry")
```

`idempotency_key` shows up in the tool schema as an optional string with that description, and the invoker sends its value as the header instead of putting it in the body. The description is the only place to tell the model what to do with it, so it should say so plainly: a new value per operation, the same value on a retry.

## What it doesn't close

The claim and the command are not in one transaction. If the process dies after the deposit commits and before the response is stored, the claim goes stale after a minute and a retry runs the deposit again. Closing that means storing the response in the same transaction as the events, which waits for the outbox work.

Keys are not scoped to a caller yet. Once the API has authentication the key has to include the caller, or two clients that pick the same key see each other's responses.

And mcp-kit can't make a model reuse the key. An agent that leaves it out gets what an HTTP client without the header gets: every call runs.

The filter is [IdempotencyFilter.cs](https://github.com/softvasco/ledger-core/blob/main/src/LedgerCore.Api/Idempotency/IdempotencyFilter.cs) in ledger-core, and the header arguments are in [mcp-kit](https://github.com/softvasco/mcp-kit). If you run a key per operation for agent traffic, I'd like to know how you scope it: per API client, per user, or per conversation.
