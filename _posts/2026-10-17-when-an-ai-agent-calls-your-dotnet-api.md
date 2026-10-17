---
title: "When an AI agent calls your .NET API"
kicker: "AI and .NET APIs / Part 1"
description: "Part 1 of a series on making an ASP.NET Core API safe to call from an AI agent through MCP: the four ways an agent differs from a normal client, and what the next eleven parts build in ledger-core and mcp-kit."
repo: mcp-kit
image: /assets/cards/when-an-ai-agent-calls-your-dotnet-api.png
---

I'm building two things that meet in the middle. [ledger-core](https://github.com/softvasco/ledger-core) is an event-sourced ledger with an ASP.NET Core minimal API in front of it. [mcp-kit](https://github.com/softvasco/mcp-kit) takes minimal API endpoints and publishes them as MCP tools, so an agent in VS Code, Cursor or a custom client can call them. The goal is that the ledger's endpoints become tools without writing them twice, and without the agent being able to do anything the HTTP API wouldn't let a person do.

This is the first part of a series about that. Each part is one problem, with the code that solves it, merged in one of the two repos. This one lists the problems.

## An agent is a strange client

Every API already has clients that retry, send bad input and misread errors. An agent does all of that more often, and it also does something a normal client never does: it decides which operation to call from text it read a moment ago. I count four differences that change how the API has to behave.

**It builds arguments from a schema, and guesses when the schema is thin.** A normal client is written by someone who read the docs. An agent gets a tool name, a description and a JSON schema, and that is all it knows about the request. The first version of mcp-kit's tool schema only had the route parameters:

```csharp
var routeParameters = tool.Endpoint.RoutePattern.Parameters.Select(p => p.Name).ToArray();
var schema = new
{
    type = "object",
    properties = routeParameters.ToDictionary(name => name, _ => new { type = "string" }),
    required = routeParameters,
};
```

For `POST /accounts/{id}/deposits` that told the model about `id` and nothing else. The amount and the currency live in the body, so the model had to guess their names from the description. Sometimes it guesses right.

**It retries.** Agent frameworks retry tool calls on timeouts, and a model will sometimes call the same tool again because it isn't sure the first call worked. For a read that is harmless. For a deposit it is a second deposit.

ledger-core already has an answer for HTTP clients: an `Idempotency-Key` header on every POST, handled by a middleware that stores the first response and replays it ([ADR-0009](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0009-idempotency-keys-for-post.md)). But mcp-kit calls the endpoint in-process, through its `RequestDelegate`, and that skips middleware. I wrote that down as a consequence in [mcp-kit's ADR-0002](https://github.com/softvasco/mcp-kit/blob/main/docs/adr/0002-tools-from-endpoint-metadata.md) before I had the idempotency middleware, and the two decisions now collide: a retried tool call would credit the account twice. Rules that must hold for tool calls have to live in endpoint filters, not middleware.

**It reads error messages as instructions.** When a tool fails, the model reads whatever came back and decides what to do next. A stack trace or "An error occurred" gives it nothing, so it tries again with different arguments. A stable code does. ledger-core answers every error with an RFC 9457 problem that carries a `code`, and mcp-kit passes the endpoint's body back as the tool error, so the model sees `account.insufficient_funds` and can tell the user instead of retrying.

**It can be talked into things.** The model decides which tool to call from its context, and its context includes text from places nobody controls: a document it was asked to summarise, a tool description from another server, the output of a previous tool. OWASP's MCP Top 10 lists tool poisoning as MCP03. For an API this means two things. A tool that changes state should not run just because the model decided to, and whatever a tool returns has to reach the model as data, not as something that reads like an instruction.

## What is already built

The pieces below are merged, or on a branch with tests:

| Problem | What the API does | Where |
|---|---|---|
| Two definitions of one operation | An endpoint opts in with `.AsMcpTool()`; the tool runs the endpoint's own `RequestDelegate`, so binding, filters and validation run as for HTTP | mcp-kit, ADR-0002 |
| Thin schemas | The input schema now comes from the request type through `JsonSchemaExporter`, merged with the route parameters, numbers kept strict | mcp-kit, branch `feat/tool-input-schema` |
| Errors the model can act on | Every error is a problem with a `code` and a `traceId`; the tool returns it with `IsError` set | ledger-core ADR-0008, mcp-kit |
| Retries over HTTP | `Idempotency-Key` on POST, stored response replayed with `Idempotent-Replayed: true` | ledger-core ADR-0009 |
| Endpoints that need auth | Refused for now, because the in-process call skips the authorization middleware | mcp-kit `EndpointToolInvoker` |

The last row is a placeholder. Until mcp-kit checks the endpoint's authorization metadata itself, it refuses to run any endpoint that has some:

```csharp
// middleware is skipped, so nothing would enforce these yet
if (endpoint.Metadata.GetMetadata<IAuthorizeData>() is not null)
{
    throw new InvalidOperationException($"Tool '{tool.Name}' requires authorization, which McpKit does not check yet.");
}
```

## The plan

| Part | Topic |
|---|---|
| 1 | The problems (this post) |
| 2 | Endpoints become tools and run in-process, and the 400 for a body that was there |
| 3 | Tool input schemas generated from the request types |
| 4 | Errors a model can read |
| 5 | Retries: idempotency that also covers tool calls |
| 6 | Testing an MCP server without an LLM |
| 7 | Read-only by default, write tools opt in |
| 8 | Authorization and scopes per tool |
| 9 | Confirmation before a tool changes state |
| 10 | Tool output as data, never as instructions |
| 11 | Audit log and rate limits per client and tool |
| 12 | The whole picture, and what I'd change |

Parts 2 to 6 are about code that exists. Parts 7 to 11 are about code I still have to write, so they come out as it gets merged. Everything is in the two repos, with the decisions in their `docs/adr` folders.

If you already expose an internal API to agents, I'd like to know which of these four bit you first.
