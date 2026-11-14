---
title: "Integration events in .NET that a consumer can rely on"
kicker: "C# and messaging / Contracts"
description: "Before the ledger publishes anything to Azure Service Bus: a contracts project with no reference to the domain, plain wire types, versions by namespace, and JSON schema snapshots that fail the build when a published shape changes."
repo: ledger-core
image: /assets/cards/versioned-integration-events-in-dotnet.png
---

The events in ledger-core's event store are private. I can change how they are written as long as I can still read the old ones, and I have done that more than once. That stops being true the moment another service reads events from a broker. A consumer deployed three months ago keeps reading the shape it was built against, and it doesn't redeploy because I renamed a property.

The ledger doesn't publish anything yet. The Service Bus topic exists in the Aspire AppHost and the API registers a sender for it, but the messages will go out through outbox-kit, and that wiring is the next piece of work. I wanted the contracts settled first, because they are the part that is hard to change later.

## Why not publish the domain events

The obvious shortcut is to serialize the domain events that are already in the store. Two things stopped me.

They use domain types. `TransferRequested` has a `Money` and a `TransferId`, so a consumer would need the domain assembly to read them, and then every change in the domain is a change for them too.

Some of them carry too little. `TransferCompleted` in the domain only has the transfer id and the time, because the process manager that handles transfers already has the accounts and the amount in its own state. A consumer doesn't have that state. If the event only has an id, the first thing every consumer does is call the API back to ask what happened, which is the coupling a message was meant to avoid.

So the published events are separate records, written for the people reading them.

## A contracts project that references nothing

`LedgerCore.Contracts` holds the published events and references nothing else in the solution. A test checks that, because it is the kind of rule that breaks quietly when someone adds a project reference to borrow one helper:

```csharp
[Fact]
public void Contracts_reference_nothing_from_the_ledger()
{
    var references = typeof(IntegrationEventAttribute).Assembly.GetReferencedAssemblies()
        .Where(reference => reference.Name!.StartsWith("LedgerCore", StringComparison.Ordinal));

    Assert.Empty(references);
}
```

Each event is a record of plain types only: `Guid`, `string`, `decimal`, `DateTimeOffset`, `int`, `long` and `bool`. Another test walks every event and fails on any property of another type. That keeps a `Money` from coming back in through a refactoring.

```csharp
namespace LedgerCore.Contracts.Transfers.V1;

// the domain event only has the id; a consumer shouldn't have to call back to learn the amount
[IntegrationEvent("ledger.transfer.completed", 1)]
public sealed record TransferCompleted(
    Guid TransferId,
    Guid SourceAccountId,
    Guid DestinationAccountId,
    decimal Amount,
    string Currency,
    DateTimeOffset OccurredAt);
```

Only three events are public for now: account opened, transfer completed and transfer failed. Deposits and withdrawals stay internal until a consumer needs them. Every event I publish is one I have to keep publishing, so I'd rather start small.

`AccountOpened` has no IBAN on purpose. A consumer that needs it can ask the API, where access is checked. Once a message is on a broker it can sit in a dead-letter queue or be copied to other subscriptions, and I have no way to take it back.

## The name on the wire

The attribute gives each event its name on the wire, separate from the C# type name:

```csharp
public sealed class IntegrationEventAttribute(string name, int version) : Attribute
{
    public string Name { get; } = name;

    public int Version { get; } = version;

    public string Type => $"{Name}.v{Version}";
}
```

`TransferCompleted` goes out as `ledger.transfer.completed.v1`. I can rename or move the record and consumers keep filtering on the same string. A test checks that no two events share a wire type, and another checks that an event with version 1 lives in a namespace ending in `.V1`, so the version in the attribute and the folder can't drift apart.

## What a new version means

The rule I wrote down in ADR 0016:

| Change | What I do |
|---|---|
| Add an optional field | Same record, same version |
| Rename, remove or change the type of a field | New record in `Transfers.V2`, publish both |
| Rename the C# type | Nothing, the wire name doesn't change |

During a migration that means two messages per event, one per version, until the last consumer of the old version has moved. It costs a bit more traffic, but no consumer fails on a shape it has never seen. The alternative, changing v1 in place and telling everyone, only works when you know who "everyone" is, and with a topic and subscriptions you often don't.

## Amounts in JSON

Amounts travel as JSON numbers from `decimal`, with the currency code as a separate field. My worry was the scale: 100.10 and 100.1 are the same number, but a ledger shows the first one, and a consumer comparing strings or printing the value would see a difference. `System.Text.Json` keeps the scale of a `decimal` both ways, and there is a test for it so a serializer change can't take it away silently:

```csharp
var sent = new TransferCompleted(Guid.NewGuid(), Guid.NewGuid(), Guid.NewGuid(), 100.10m, "EUR", DateTimeOffset.UnixEpoch);

var received = JsonSerializer.Deserialize<TransferCompleted>(JsonSerializer.Serialize(sent));

Assert.Equal("100.10", received!.Amount.ToString(System.Globalization.CultureInfo.InvariantCulture));
```

A consumer in JavaScript would still read it into a double. That is their side, but it is one more reason the currency travels with the amount, so they at least know how many decimals to show.

## Schema snapshots

The rule tests check the shape of the rules, not each field. Nothing above stops someone from renaming `SourceAccountId` to `FromAccountId` inside V1, and that is exactly the change that breaks a consumer.

For that, every published event has a JSON schema checked into the test project, `Schemas/ledger.transfer.completed.v1.json`. .NET 9 added `JsonSchemaExporter`, so the test can generate the schema from the type with the same serializer options the publisher will use and compare it with the file:

```csharp
[Theory]
[MemberData(nameof(AllEvents))]
public void A_published_event_keeps_its_schema(Type type)
{
    var name = type.GetCustomAttribute<IntegrationEventAttribute>()!.Type;
    var actual = Options.GetJsonSchemaAsNode(type).ToJsonString(Options) + "\n";
    var path = Path.Join(SchemaDirectory(), $"{name}.json");
    var expected = File.Exists(path) ? File.ReadAllText(path).ReplaceLineEndings("\n") : null;

    if (actual != expected)
    {
        File.WriteAllText(Path.ChangeExtension(path, ".received.json"), actual);
    }

    Assert.Equal(expected, actual);
}
```

When it fails it writes a `.received.json` next to the snapshot, which is git-ignored. If the change was meant (an optional field added), I copy it over the snapshot and the diff shows up in the pull request where a reviewer can see it. If it wasn't meant, the build is red before anything reaches a broker. A second test fails when a snapshot has no event any more, so deleting a published event also has to be a visible decision.

The options set `RespectNullableAnnotations` and `RespectRequiredConstructorParameters`, so the schema lists every constructor parameter as required and a non-nullable `string` stays non-nullable. Without them every field comes out optional, which says the opposite of what the record does.

The first version built the snapshot path with `Path.Combine`, and CodeQL flagged it: if the second argument is ever an absolute path, `Path.Combine` drops everything before it. Here the event name can't be one, but `Path.Join` doesn't have that behaviour at all, so I switched rather than argue with the alert.

## What comes next

The mapping from domain events to these contracts is still to write, and it goes with the publisher, because `TransferCompleted` needs the accounts and the amount from the process manager's state. The publisher will write to outbox-kit's table in the same transaction as the events, and the relay sends them to the `ledger.events` topic.

The code is in [ledger-core](https://github.com/softvasco/ledger-core), under `src/LedgerCore.Contracts` and `tests/LedgerCore.Contracts.Tests`, with the reasoning in ADR 0016.
