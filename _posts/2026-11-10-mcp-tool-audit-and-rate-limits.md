---
title: "Auditing and rate limiting MCP tool calls in ASP.NET Core"
kicker: "AI and .NET APIs / Part 11"
description: "Part 11 of the series on making an ASP.NET Core API safe for AI agents: one audit record per tool call with a hash of the arguments instead of their values, and a limit per client and tool that the normal rate limiting middleware can't give you."
repo: mcp-kit
image: /assets/cards/mcp-tool-audit-and-rate-limits.png
---

When an agent does something odd through your API, the first questions are always the same: which tool did it call, as whom, with what, and what came back. And when an agent gets stuck in a loop, it calls the same tool again and again, often with the same arguments, much faster than any person clicking a button.

[Part 9](/2026/11/mcp-tool-confirmation-without-server-state/) made a write wait for a second call. This part covers the two things around every call in [mcp-kit](https://github.com/softvasco/mcp-kit): a record of it, and a limit on how often it can happen.

## Why the API's own logs and limits don't cover it

The endpoint logs answer part of "what happened", but they don't know the request came from a tool. A confirmation request (Part 9) never reaches the endpoint at all, so it leaves nothing there.

Rate limiting has a worse problem. ASP.NET Core has rate limiting middleware, but every MCP request goes to the one `/mcp` route. A limiter on that route sees one endpoint and can't tell `get_balance` from `transfer`. And a policy on the endpoint itself (`RequireRateLimiting`) never runs, because the tool calls the endpoint's request delegate in-process, skipping the middleware pipeline. It's the same reason the idempotency check in [Part 5](/2026/10/idempotency-keys-for-agent-retries/) had to be a filter.

So both checks live inside the tool call.

## One record per call

Every endpoint tool call produces one record ([ADR 0008](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0008-audit-every-tool-call.md)):

```csharp
public sealed record ToolCallRecord(
    string Tool,
    string Caller,
    string ArgumentsHash,
    ToolCallOutcome Outcome,
    int? StatusCode,
    int ResultBytes,
    TimeSpan Duration,
    DateTimeOffset StartedAt);
```

The outcome is one of `Succeeded`, `Failed`, `ConfirmationRequested`, `ConfirmationRefused` and `RateLimited`. The status code is only there when the endpoint actually ran. If the endpoint throws, the call is recorded as `Failed` with no status, and the exception goes on up.

The caller is the name identifier claim, then `sub`, then the identity name. Duration comes from `TimeProvider`, so tests can check it without timing anything.

The default writes one structured log entry per call, event `ToolCalled`. A host that wants an audit table registers its own `IToolCallAudit` before calling `AddMcpKit()`, which only adds the logger one if nothing is there yet:

```csharp
public interface IToolCallAudit
{
    ValueTask RecordAsync(ToolCallRecord toolCall, CancellationToken cancellationToken);
}
```

The audit runs inside the call, so a slow implementation slows every tool. Anything that writes to a database should put the record on a queue and return.

## A hash instead of the arguments

The obvious version logs the arguments. I didn't, because here the arguments are account ids, names and amounts. Copying them into a log that is kept for months and readable by everyone with log access is worse than the problem it solves.

What you usually want from the arguments is to compare calls: did the agent send the same transfer twice? A hash answers that without holding the values. It's the same fingerprint the confirmation token from Part 9 uses:

```csharp
// key order is the client's choice, so sort before hashing; values are compared as compact JSON
public static string Arguments(IReadOnlyDictionary<string, JsonElement> arguments)
{
    var canonical = new StringBuilder();
    foreach (var (name, value) in arguments.OrderBy(a => a.Key, StringComparer.Ordinal))
    {
        canonical.Append(name).Append('=').Append(JsonSerializer.Serialize(value)).Append('\n');
    }

    return Convert.ToBase64String(SHA256.HashData(Encoding.UTF8.GetBytes(canonical.ToString())));
}
```

The `confirmation_token` argument is left out before hashing, so the preview call and the confirmed call of the same write have the same hash and show up as a pair.

A plain SHA-256 is not encryption. An account id and a small amount have few possible values, and someone who wants them can hash candidates until one matches. It keeps the values out of casual reading and out of log search. If that isn't enough, the next step is an HMAC with a key the host owns, which means a key to manage. I wrote that down in the ADR rather than adding it before anyone needs it.

## A limit per client and tool

The host sets one limit for every tool, and an endpoint can replace it with its own ([ADR 0009](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0009-rate-limits-per-client-and-tool.md)):

```csharp
builder.Services.AddMcpKit(options => options.RateLimit = new ToolRateLimit(60, TimeSpan.FromMinutes(1)));

app.MapPost("/transfers", Transfer)
    .AsMcpWriteTool()
    .WithMcpRateLimit(5, TimeSpan.FromMinutes(1));
```

There is no default. Ten calls a minute is fine for a transfer and far too low for a lookup the agent makes for every row of a statement, so the number has to come from whoever knows the API.

The key is the caller plus the tool name. A refused call doesn't reach the endpoint and gets back an error the model can act on:

```json
{"title":"Too many calls to this tool. Wait before calling it again.","code":"tool.rate_limited","retry_after_seconds":30}
```

Refused calls still count, so an agent that keeps hammering doesn't get in sooner by trying harder.

## Why not System.Threading.RateLimiting

The counter is my own, a fixed window per key in a `ConcurrentDictionary`, about 40 lines. The BCL limiters replenish on their own timer, which means a test has to sleep to see a window end. With the counter on `TimeProvider`, the test moves a `FakeTimeProvider` instead:

```csharp
await client.CallToolAsync("get_balance", Account, cancellationToken: Token);
await client.CallToolAsync("get_balance", Account, cancellationToken: Token);
_clock.Advance(TimeSpan.FromSeconds(30.5));
var refused = await client.CallToolAsync("get_balance", Account, cancellationToken: Token);

Assert.True(refused.IsError);
var body = JsonDocument.Parse(Assert.IsType<TextContentBlock>(Assert.Single(refused.Content)).Text).RootElement;
Assert.Equal("tool.rate_limited", body.GetProperty("code").GetString());
Assert.Equal(30, body.GetProperty("retry_after_seconds").GetInt32());
Assert.Equal(2, _balanceReads);
```

The last line matters most. The endpoint counts its own calls, and the count stays at 2. A refusal that still ran the handler would look identical from the client's side.

The update inside `AddOrUpdate` can run more than once when two calls race, so it never changes a window in place; it returns a new one. Windows nobody has touched since they ended are dropped once a minute, so memory follows the number of active callers.

## Where they sit in the call

The order inside a tool call is: audit around everything, then the rate limit, then argument validation, then confirmation, then the endpoint. Two consequences of that order are easy to miss.

A write that needs confirmation takes two calls, and both count against the limit. With `WithMcpRateLimit(5, ...)` on a confirmed transfer, the agent gets two full transfers a minute, plus one preview. Set the number with that in mind.

And every refusal is in the audit log. Rate limited calls, previews and bad tokens all leave a record, so you can see an agent stuck in a loop before it hits anything real.

## What it doesn't do

Counts live in memory on each instance. With three instances behind a load balancer, a client can make up to three times the limit. That still caps a runaway loop, which is what it's for. An exact global limit needs a shared store behind the same check.

All unauthenticated callers share one count per tool, because they all have the same empty caller id. Fixed windows also let a client make twice the limit across a window boundary. A sliding window or token bucket could replace the counter without changing the options.

Hand-written SDK tools don't go through the endpoint tool, so they are neither audited nor limited.

The code is [AuditedToolCall.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/AuditedToolCall.cs), [ToolRateLimiter.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/ToolRateLimiter.cs) and [EndpointMcpServerTool.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/EndpointMcpServerTool.cs) in mcp-kit. If you run an MCP server on several instances, did you need an exact global limit, or was a per-instance cap enough?
