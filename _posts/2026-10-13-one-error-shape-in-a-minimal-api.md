---
title: "One error shape for an ASP.NET Core minimal API"
kicker: "ASP.NET Core / APIs"
description: "Every error from the ledger-core API is now an RFC 9457 problem with a stable code, a traceId, and a status picked from one table. How it's wired in .NET 10 and why some refusals are 409 and others 422."
repo: ledger-core
image: /assets/cards/one-error-shape-in-a-minimal-api.png
---

Until yesterday the [ledger-core](https://github.com/softvasco/ledger-core) API answered errors in three different shapes. A bad IBAN came back as `ValidationProblemDetails`. A refused command, like a withdrawal bigger than the balance, was a 409 with the raw domain error serialized as `{ code, message }`. Not found was an empty 404, the same as an unknown route. And anything thrown below the endpoint came back as a 500.

That last one was a bug a client would hit on day one. A deposit of 0 passed the endpoint's checks, the command validator rejected it, nothing caught the exception, and the client got a 500 for its own mistake.

So now every error is an RFC 9457 problem (`application/problem+json`), and a client branches on one field. The decision is in [ADR-0008](https://github.com/softvasco/ledger-core/blob/main/docs/adr/0008-problem-details-for-every-error.md). Here is what it takes in .NET 10 and the few choices I had to make.

## What a client sees now

These are real responses from the API running locally, against a fresh EUR account with nothing in it:

```text
POST /accounts/{id}/withdrawals  {"amount":10,"currency":"EUR"}
409 {"title":"Conflict","status":409,
     "detail":"Account 01a136b8-... has EUR 0.00, less than the EUR 10.00 requested.",
     "code":"account.insufficient_funds","traceId":"00-38d2d75d..."}

POST /accounts/{id}/deposits  {"amount":10,"currency":"USD"}
422 {"title":"Unprocessable Entity","status":422,
     "detail":"Account 01a136b8-... is in EUR, not USD.",
     "code":"account.currency_mismatch","traceId":"00-f005bad2..."}

POST /accounts/{id}/deposits  {"amount":0,"currency":"EUR"}
400 {"title":"One or more validation errors occurred.","status":400,
     "errors":{"amount":["The amount must be greater than zero."]},"traceId":"00-ea1d9990..."}

GET /nowhere
404 {"title":"Not Found","status":404,"traceId":"00-f2343a74..."}
```

I trimmed the `type` field, which is the default link to the RFC section for that status. The parts that matter are `code`, which a program reads, `detail`, which a person reads, and `traceId`, which support reads.

## The wiring

Three lines in `Program.cs` do most of the work:

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<ProblemExceptionHandler>();
// ...
app.UseExceptionHandler();
app.UseStatusCodePages();
```

`AddProblemDetails` registers `IProblemDetailsService`, and everything that writes an error goes through it. That's where the `traceId` comes from, so I never add it by hand. `UseStatusCodePages` covers responses that have a status and no body, which is what routing gives you for an unknown path or a wrong method. Without it the 404 for `/nowhere` is empty.

`UseExceptionHandler` with no arguments runs the registered `IExceptionHandler`s in order. Mine only answers the exceptions a client can do something about:

```csharp
var problem = exception switch
{
    CommandValidationException invalid => Invalid(invalid),
    ConcurrencyConflictException => new ProblemDetails
    {
        Status = StatusCodes.Status409Conflict,
        Detail = "The account changed while the request ran. Sending it again is safe.",
        Extensions = { [DomainProblems.CodeExtension] = ConcurrencyConflictCode },
    },
    BadHttpRequestException badRequest => new ProblemDetails { Status = badRequest.StatusCode },
    _ => null,
};
```

For anything else it returns `false`, and the built-in handler writes a generic 500. The exception message never goes out. There's a test that makes the event store throw `InvalidOperationException("connection string has a password")` and checks the body doesn't contain "password". The details stay in the logs, and the `traceId` in the response is how you find them.

`CommandValidationException` comes from the validation step that runs before every command handler. It carries every error, not only the first one, so the handler groups them by field and the client gets the same `errors` dictionary it would get from endpoint validation. A client doesn't need to know which layer refused the input.

## Domain errors and their status codes

Business rules in ledger-core don't throw. The aggregate returns a `Result` with a `DomainError` that has a code and a message. The endpoint turns a failed result into a problem with one extension method, and one dictionary decides the status:

```csharp
// 409 when the request would work against another state of the account, 422 when it can never work as sent
private static readonly Dictionary<string, int> StatusByCode = new(StringComparer.Ordinal)
{
    [AccountErrors.NotFoundCode] = StatusCodes.Status404NotFound,
    [AccountErrors.InvalidStateCode] = StatusCodes.Status409Conflict,
    [AccountErrors.InsufficientFundsCode] = StatusCodes.Status409Conflict,
    [AccountErrors.CurrencyMismatchCode] = StatusCodes.Status422UnprocessableEntity,
    [LedgerErrors.TooFewPostingsCode] = StatusCodes.Status422UnprocessableEntity,
    [LedgerErrors.UnbalancedCode] = StatusCodes.Status422UnprocessableEntity,
};
```

The 409 versus 422 split was the one real decision here. The rule I went with:

| Status | Meaning | Examples |
|---|---|---|
| 400 | the request is malformed or fails input rules | amount 0, unknown currency, too many decimals |
| 409 | the same request can succeed after the account changes | insufficient funds, frozen account |
| 422 | the request can never succeed as sent | deposit in USD to a EUR account, unbalanced entry |

A client can act on that difference. A 409 means retrying later might work, maybe after a deposit. A 422 means the request itself has to change. The `concurrency_conflict` 409 fits the same rule: someone else wrote to the account first, and sending the same request again is safe.

A code that isn't in the table gets 422. When I add a new domain rule and forget the row, the client still gets a sensible "your request can't work" instead of a 500.

## The type field

RFC 9457 suggests a URI in `type` that identifies the problem, ideally a page explaining it. I left the default. A URI per code only helps if there's a page behind it, and there isn't one yet. The `code` extension is the contract: short, stable, namespaced by aggregate (`account.`, `ledger.`), and the same string the domain uses. When the API docs exist, `type` can point into them without changing what clients branch on.

## The breaking change

Overdrawing was already a 409 before this change. Its body went from `{ code, message }` to a problem with `code` and `detail`, which breaks any client reading `message`. Nothing outside the tests calls the API yet, so I took the break now. After the first release that would need a version bump.

## Next

These codes matter beyond HTTP. ledger-core's commands are about to be exposed as MCP tools in [mcp-kit](https://github.com/softvasco/mcp-kit), and an agent calling a tool needs the same thing a client does: a code it can branch on and a detail it can show. The next step on the API side is the OpenAPI document, so the problem responses are listed per endpoint instead of only being described here.

The code is in [`src/LedgerCore.Api/Problems`](https://github.com/softvasco/ledger-core/tree/main/src/LedgerCore.Api/Problems), and the tests that pin each status are in [`ProblemResponseTests.cs`](https://github.com/softvasco/ledger-core/blob/main/tests/LedgerCore.Api.Tests/ProblemResponseTests.cs).
