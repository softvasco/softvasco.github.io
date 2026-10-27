---
title: "MCP tools from an ASP.NET Core API: read-only unless you say otherwise"
kicker: "AI and .NET APIs / Part 7"
description: "Part 7 of the series on making an ASP.NET Core API safe for AI agents: why reads and writes get different markers, why a POST marked as a read fails at startup, and why the host has to switch write tools on as well."
repo: mcp-kit
image: /assets/cards/mcp-tools-read-only-by-default.png
---

When an endpoint becomes an MCP tool, a model decides when to call it. For `get_account` that is fine. A wrong call costs one request and some tokens. For `deposit` it is a different kind of mistake: money moves, and the reason may be a sentence the model read in a tool result or a document, not something the user asked for. [Part 5](/2026/10/idempotency-keys-for-agent-retries/) dealt with retries of writes. This part is about which writes an agent can see at all.

## One marker was not enough

The first version of [mcp-kit](https://github.com/softvasco/mcp-kit) had a single opt-in. An endpoint became a tool only with `.AsMcpTool()`, and nothing was exposed without it ([ADR 0002](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0002-tools-from-endpoint-metadata.md)). That is better than exposing every route, but a deposit and a balance read carried the same marker. McpKit couldn't tell them apart, so neither could the agent. And putting `.AsMcpTool()` on a route group, which is the natural thing to do in a minimal API, would publish every write in the group along with the reads.

So I split it in two ([ADR 0004](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0004-read-only-by-default.md)):

```csharp
app.MapGet("/accounts/{id}", (string id) => Results.Ok(new { id }))
    .WithName("GetAccount")
    .AsMcpTool();

app.MapPost("/accounts/{id}/deposits", (string id, DepositRequest deposit) => /* ... */)
    .WithName("Deposit")
    .WithSummary("Deposit money into an account")
    .AsMcpWriteTool();
```

`AsMcpTool()` means a read. `AsMcpWriteTool()` means the endpoint changes state. Both only add metadata to the endpoint; the checks happen when McpKit builds its tool list.

## A POST marked as a read fails at startup

The only signal McpKit has about whether an endpoint reads or writes is the HTTP verb. It can't look inside the handler. So the catalog that turns endpoints into tools refuses the obvious mistake:

```csharp
// the verb is the only hint we have; a POST behind AsMcpTool is almost always a mistake
if (!marker.ChangesState && !HttpMethods.IsGet(methods[0]) && !HttpMethods.IsHead(methods[0]))
{
    throw new InvalidOperationException(
        $"Endpoint '{methods[0]} {route}' changes state, so it can't be a read-only tool. Use AsMcpWriteTool if agents should call it.");
}
```

This runs once, when the MCP server is set up, together with the other checks on the markings: an endpoint that maps more than one verb, a tool with no name, two tools with the same name. I wanted all of these to fail when the app starts, in front of whoever made the change, and not on the first tool call in some environment nobody is watching.

It can't catch the opposite case. A GET that changes state, like a `/confirm?token=...` link, looks like a read to McpKit. Whoever wrote that endpoint has to mark it with `AsMcpWriteTool()`. A GET with side effects is a problem for crawlers and link previews anyway, so it is worth fixing at the endpoint rather than in McpKit.

There is a false positive too. A search that takes its filters in a POST body only reads, but McpKit will refuse it as a read-only tool. For now it has to be a write tool. I'd rather have a read that has to be marked as a write than a write that gets through as a read.

## The host has to allow writes too

Marking an endpoint is the API author's decision. Whether agents may change anything in a given deployment is the host's decision, and the two are often different people, or the same API running in two places. So there is a second switch in the options:

```csharp
// off by default: a host exposes reads only until someone decides agents may change things
public bool AllowWriteTools { get; set; }
```

When it is false, write tools are not registered with the MCP server at all:

```csharp
var tools = provider.GetRequiredService<EndpointToolCatalog>().Discover()
    .Where(t => options.AllowWriteTools || !t.ChangesState);
```

They are not just missing from `tools/list`. There is no tool behind the name, so a client that guesses `deposit` and calls it gets nothing to run. The same API can have a read-only deployment for an assistant that answers questions about balances and another one that lets an agent move money, and the difference is one line in the host's setup:

```csharp
builder.Services.AddMcpKit(options => options.AllowWriteTools = true);
```

That line is easy to find in a review, and easy to search for across deployments. The default for a new host is reads only.

The test for it starts a host with one read, one write and one endpoint that isn't a tool:

```csharp
[Fact]
public async Task Write_tools_are_not_listed_unless_the_host_allows_them()
{
    await using var app = await StartHost(allowWriteTools: false);
    await using var client = await Connect(app);

    var tools = await client.ListToolsAsync(cancellationToken: Token);

    Assert.Equal(["get_account"], tools.Select(t => t.Name));
}
```

## Annotations come from the same split

MCP has tool annotations for this: `readOnlyHint`, `destructiveHint` and `idempotentHint`. A client can use them to decide when to ask the user before a call. McpKit fills them in from the marker and the verb, so nobody writes them by hand and they can't drift from what the endpoint does:

| Endpoint | readOnlyHint | destructiveHint | idempotentHint |
|---|---|---|---|
| GET or HEAD, `AsMcpTool()` | true | | |
| POST or PATCH, `AsMcpWriteTool()` | false | false | false |
| PUT or DELETE, `AsMcpWriteTool()` | false | true | true |

```csharp
private static ToolAnnotations Annotate(EndpointTool tool) => tool.ChangesState
    ? new ToolAnnotations
    {
        ReadOnlyHint = false,
        DestructiveHint = HttpMethods.IsDelete(tool.HttpMethod) || HttpMethods.IsPut(tool.HttpMethod),
        IdempotentHint = HttpMethods.IsDelete(tool.HttpMethod) || HttpMethods.IsPut(tool.HttpMethod),
    }
    : new ToolAnnotations { ReadOnlyHint = true };
```

The spec is clear that these are hints, and that a client should not trust them from a server it doesn't trust. That is why they are not the protection here. They help a well-behaved client show a confirmation prompt. The two switches above are what keep a write away from an agent that shouldn't have it.

One row looks wrong at first. The deposit endpoint from Part 5 takes an `Idempotency-Key`, and it is still annotated as not idempotent. The hint describes the endpoint, and the key only helps when the agent actually sends it. An agent that retries without it deposits twice, so the honest answer is false.

## What this doesn't decide

The split says whether an agent can see a write tool. It doesn't say which caller may use it, or whether a person should approve the call first. Those are the next parts: scopes per tool, and a confirmation step that answers the first call with a preview and a token. Both build on knowing which tools change state, which is why this came first.

The catalog is [EndpointToolCatalog.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/EndpointToolCatalog.cs) in mcp-kit. If you expose an API to agents, do you run separate read-only and read-write deployments, or one deployment with scopes deciding everything?
