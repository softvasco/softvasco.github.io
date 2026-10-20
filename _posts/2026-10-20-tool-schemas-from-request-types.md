---
title: "MCP tool schemas from the request types you already have"
kicker: "AI and .NET APIs / Part 3"
description: "Part 3 of the series on making an ASP.NET Core API safe for AI agents: building each MCP tool's input schema from the endpoint's body type, route and query parameters with JsonSchemaExporter, and the number handling that makes the schema say string."
repo: mcp-kit
image: /assets/cards/tool-schemas-from-request-types.png
---

[Part 1](/2026/10/when-an-ai-agent-calls-your-dotnet-api/) listed four ways an agent differs from a normal API client. The first one was that it builds arguments from a JSON schema and guesses when the schema is thin. This part is about how [mcp-kit](https://github.com/softvasco/mcp-kit) builds that schema now.

## What the model saw before

mcp-kit turns a minimal API endpoint into an MCP tool when it is marked with `.AsMcpTool()` or `.AsMcpWriteTool()`. The tool runs the endpoint's own `RequestDelegate` in-process, so binding and validation are the endpoint's. The schema the model gets is a different matter. The first version only looked at the route:

```csharp
var routeParameters = tool.Endpoint.RoutePattern.Parameters.Select(p => p.Name).ToArray();
```

For `POST /accounts/{id}/deposits` the model was told about `id` and nothing else. The amount and the currency are in the body. The model could only find their names in the tool description, if I had written them there. If it sent `value` instead of `amount`, the serializer ignored the unknown property and the deposit arrived with an amount of zero.

Writing the schema by hand for each tool would fix it, but then there are two definitions of the same request, and they drift. The endpoint already declares everything the schema needs. It just declares it in three places.

## Where the arguments come from

A tool call is one flat JSON object of arguments. The endpoint takes its input from up to three sources: the route, the query string and the body. So the schema is a merge of three parts.

**The body.** When a minimal API handler has a complex parameter, the endpoint gets an `IAcceptsMetadata` with the request type. Since .NET 9, `System.Text.Json` can export a JSON schema for any type through `JsonSchemaExporter`, with the same naming and nullability rules the serializer uses. So the body's properties come straight from the type:

```csharp
var bodyType = endpoint.Metadata.GetMetadata<IAcceptsMetadata>()?.RequestType;
if (bodyType is not null && Json.GetJsonSchemaAsNode(bodyType) is JsonObject body)
{
    foreach (var (name, property) in body["properties"]?.AsObject() ?? [])
    {
        properties[name] = property?.DeepClone();
    }

    required.AddRange(body["required"]?.AsArray().Select(n => n!.GetValue<string>()) ?? []);
}
```

The `required` list is the exporter's too. For `record Deposit(decimal Amount, string Currency, string? Reference = null)` it lists `amount` and `currency`, and leaves `reference` out because the constructor parameter has a default.

**The route.** Route parameters come from the route pattern, and their types from the handler's parameters with the same name. An `int id` becomes `integer`, a `Guid` becomes a string with format `uuid`. A route value is required unless the pattern marks it optional (`{page?}`). Route values go in after the body, so if the body has a property with the same name, the route wins, the same way the endpoint sees it.

**The query string.** For a GET, every handler parameter that is not in the route and that minimal APIs would bind from the query string becomes an argument. That rule has a few parts: a parameter marked `[FromQuery]` uses its query name, a simple type (anything with `TryParse`, enums, arrays of those) binds from the query by default, and anything marked `[FromServices]`, `[FromHeader]` or `[AsParameters]`, or `HttpContext` and `CancellationToken`, is not an argument at all. A query parameter is required when it has no default value and is not nullable.

One test covers the "not an argument" side:

```csharp
var schema = SchemaOf(app => app.MapGet("/search", (
    string q,
    [FromHeader(Name = "X-Tenant")] string? tenant,
    [FromServices] TimeProvider time,
    HttpContext context,
    CancellationToken cancellationToken) => q));

Assert.Equal(["q"], schema.GetProperty("properties").EnumerateObject().Select(p => p.Name));
```

Types bound through `TryParse` are a special case. An `AccountNumber` record that implements `IParsable<AccountNumber>` has a JSON shape of an object, but it arrives as text in the URL. The schema says `string` for any route or query type whose exported schema is an object.

## The number that came out as a string

The first run of the body schema gave me this for `amount`:

```json
"amount": { "type": ["string", "number"], "pattern": "^-?(?:0|[1-9]\\d*)(?:\\.\\d+)?$" }
```

The exporter is right. I had used `JsonSerializerOptions.Web`, the same defaults minimal APIs use, and those include `JsonNumberHandling.AllowReadingFromString`. The serializer really does accept `"25"` for a decimal, so the schema says so.

For a model it reads as an invitation to send `"25"`, and nothing is gained by allowing it. A schema should ask for the one form I want. So the schema builder uses its own options: web naming, strict numbers.

```csharp
// web naming like the endpoint, but numbers as numbers: a model should not send "25" for 25
private static JsonSerializerOptions CreateOptions()
{
    var options = new JsonSerializerOptions(JsonSerializerOptions.Web)
    {
        NumberHandling = JsonNumberHandling.Strict,
        TypeInfoResolver = new DefaultJsonTypeInfoResolver(),
    };
    options.MakeReadOnly();
    return options;
}
```

With those options the same property comes out as `{ "type": "number" }`. The endpoint still reads with the web defaults, so a string that slips through still binds. The schema is only stricter about what it asks for.

The full schema for the deposit tool is now:

```json
{
  "type": "object",
  "properties": {
    "amount": { "type": "number" },
    "currency": { "type": "string" },
    "reference": { "type": ["string", "null"], "default": null },
    "id": { "type": "string" }
  },
  "required": ["amount", "currency", "id"]
}
```

## Headers

Some arguments are headers. `Idempotency-Key` is the one that matters in this series, and Part 5 is about it. A schema built from the handler would never show it, because the idempotency filter reads the header, not the handler. So an endpoint declares it with `.WithMcpToolHeader("Idempotency-Key", "idempotency_key", "...")`. It shows up as an optional string argument with that description, and the invoker moves it into the request header. If a header argument has the same name as another argument, startup fails, because the model would have no way to say which one it meant.

## What this does not cover

Form fields and file uploads are not tool arguments, since a tool call is JSON. `[AsParameters]` is skipped for now, so an endpoint that uses it gets a schema without those parameters, and I will need to expand it into its members. Validation attributes like `[Range]` are not in the schema either: the exporter does not read DataAnnotations, so the endpoint's validation is still the only check, and its error is a problem with a code (Part 4).

The code is in [ToolInputSchema.cs](https://github.com/softvasco/mcp-kit/blob/main/src/McpKit/ToolInputSchema.cs), with the tests next to it in `ToolInputSchemaTests`. If you generate tool schemas for an API another way, from OpenAPI or by hand, I'd like to hear which one drifted less.
