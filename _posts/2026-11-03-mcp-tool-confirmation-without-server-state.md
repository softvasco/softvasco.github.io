---
title: "Making an MCP tool in ASP.NET Core wait for a yes, with no state on the server"
kicker: "AI and .NET APIs / Part 9"
description: "Part 9 of the series on making an ASP.NET Core API safe for AI agents: a write tool that answers the first call with a preview and a token, runs only on a second call with the same arguments, and keeps nothing on the server in between."
repo: mcp-kit
image: /assets/cards/mcp-tool-confirmation-without-server-state.png
---

Some writes shouldn't run the moment a model decides to call them. A transfer, closing an account, anything a person would want to look at first. [Part 7](/2026/10/mcp-tools-read-only-by-default/) decided which write tools an agent can see at all, and per-tool authorization ([ADR 0005](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0005-authorization-per-tool.md)) decides which caller may use each one. Both answer "may this caller use this tool?". Neither says anything about one particular call, with these arguments, right now.

This part adds that step to [mcp-kit](https://github.com/softvasco/mcp-kit): the first call to a tool runs nothing and returns what would run, and only a second call runs the endpoint.

## Why not elicitation

MCP already has a way for a server to ask the user something: elicitation. In the middle of a tool call, the server sends a request back to the client, the client shows a prompt, and the answer comes back on the same session. That needs a stateful session, because the server has to hold the open call while it waits.

McpKit runs stateless by default, so the host scales out like any other ASP.NET Core API behind a load balancer. And many clients don't support elicitation yet. So the confirmation had to work as two plain tool calls, and the server couldn't remember anything between them ([ADR 0006](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0006-confirmation-before-a-write.md)).

## Two calls

An endpoint opts in with one extension method, on top of being a write tool:

```csharp
app.MapPost("/accounts/{id}/deposits", (string id, DepositRequest deposit) => /* ... */)
    .WithName("Deposit")
    .AsMcpWriteTool()
    .RequireMcpConfirmation(TimeSpan.FromMinutes(1));
```

The lifetime defaults to 5 minutes. The tool's input schema gets an optional `confirmation_token` argument, so the model knows it exists. A call without it doesn't reach the endpoint. It gets this back:

```csharp
// not an error: the call did what it should, which is nothing until someone says yes
private CallToolResult Pending((string Token, DateTimeOffset ExpiresAt) issued, Dictionary<string, JsonElement> call) =>
    Text(JsonSerializer.Serialize(new Dictionary<string, object>
    {
        ["status"] = "confirmation_required",
        ["message"] = "Nothing has run. Show the user this call and, if they agree, call again with the same arguments and confirmation_token.",
        ["tool"] = tool.Name,
        ["arguments"] = call,
        [McpToolConfirmationMetadata.Argument] = issued.Token,
        ["expires_at"] = issued.ExpiresAt,
    }), isError: false);
```

The `arguments` field is what the server read, not what the model meant to send. If the model got the account id wrong, that is visible in the preview before anything happens.

I went back and forth on whether this first answer should be an error. A tool error tells the model something went wrong, and models tend to react to that by trying something else. Nothing went wrong here. The tool did what it was told to do, so `isError` is false and the message says what to do next.

The second call sends the same arguments plus the token. McpKit checks the token, removes it from the arguments, and runs the endpoint as usual. The endpoint never sees `confirmation_token`, so a deposit handler doesn't need to know confirmation exists.

## What the token carries

With no state on the server, the token has to carry everything needed to check it later. It is four things, sealed with ASP.NET Core data protection:

```csharp
var payload = new Payload(tool, CallFingerprint.Caller(user), CallFingerprint.Arguments(arguments), expiresAt.ToUnixTimeMilliseconds());
return (_protector.Protect(JsonSerializer.Serialize(payload)), expiresAt);
```

The tool name, the caller's id (name identifier, then `sub`, then the name), a SHA-256 of the arguments, and the expiry. Data protection encrypts and signs it, so a client can't read the hash out or build a token of its own. Nothing is written to a database or a cache.

The arguments hash is the part that matters most. Without it, a model could preview a deposit of 25, the user would say yes to 25, and the model could send 2500 with the same token. With it, the check compares the hash in the token to the hash of the arguments on the second call:

```csharp
if (payload is null
    || payload.Tool != tool
    || payload.Caller != CallFingerprint.Caller(user)
    || payload.Arguments != CallFingerprint.Arguments(arguments))
{
    return ConfirmationCheck.Invalid;
}
```

Any difference gives `tool.confirmation_invalid`, and an old token gives `tool.confirmation_expired`. Both are tool errors, both run nothing, and both tell the model to call again without a token to get a fresh one.

Hashing JSON needs a canonical form, because key order is up to the client:

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

So `{"a":1,"b":2}` and `{"b":2,"a":1}` match. `25` and `25.0` don't. I left that as it is. A model that previews 25 and then sends 25.0 has changed what it sends, and asking for a new token costs one extra call.

The audit log records the same argument hash and the rate limiter keys on the same caller id, which is why both live in one small class.

## The tests

The tests run a real MCP client against a test server, with a `FakeTimeProvider` for the clock. The one I care most about:

```csharp
[Fact]
public async Task A_token_for_one_amount_does_not_confirm_another()
{
    await using var app = await StartHost();
    await using var client = await Connect(app);
    var token = await PreviewAsync(client, 25m);

    var result = await client.CallToolAsync("deposit", Args(2500m, token), cancellationToken: Token);

    Assert.True(result.IsError);
    Assert.Equal("tool.confirmation_invalid", Json(result).GetProperty("code").GetString());
    Assert.Equal(0, _deposits.Count);
}
```

The others check that the first call leaves the deposit count at zero, that the second call runs the endpoint exactly once, and that a token with a one minute lifetime fails after the clock moves two minutes. Because the expiry comes from `TimeProvider`, that test doesn't sleep.

## Where it sits in the pipeline

Order matters here. Confirmation runs after the SDK's authorization filter, so a caller who may not use the tool never gets a token for it. Inside McpKit the rate limit comes first, so asking for previews over and over counts against the limit like any other call. And every outcome goes to the audit log: a preview is recorded as `ConfirmationRequested`, a bad or old token as `ConfirmationRefused`, so you can see how often agents stop at the preview.

## What it doesn't prove

Nothing stops a model from taking the token and confirming on its own. The token makes the step explicit and gives the client a clear point to ask the user, but it is not proof that a person said yes. If a host needs that proof, it has to use elicitation on a stateful server, or confirm outside the agent, in its own UI.

There is also a deployment cost. Data protection keys must be shared between instances, or a token issued by one is invalid on the next. In production that means a key ring in shared storage, the same setup cookie authentication already needs, so most ASP.NET Core apps behind a load balancer have it.

The code is [ToolConfirmations.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/ToolConfirmations.cs) and [EndpointMcpServerTool.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/EndpointMcpServerTool.cs) in mcp-kit. Has anyone shipped elicitation to real users yet, and which clients did you have to support?
