---
title: "Why 0.05 × 0.5 is 0.02 in my ledger"
kicker: "C# / decimal"
description: "decimal.Round uses banker's rounding by default. How ledger-core uses it, and why amounts with too many decimals are rejected instead of rounded."
repo: ledger-core
image: /assets/cards/rounding-in-a-ledger.png
---

In [ledger-core](https://github.com/softvasco/ledger-core), `Money.Of(0.05m, Currency.Eur).Multiply(0.5m)` returns EUR 0.02. The exact result is 0.025, which can't be stored in euros, and it rounds down to 0.02 instead of up to 0.03. Here's why, plus a few related rules in the same type.

## decimal.Round rounds half to even

`decimal.Round` uses `MidpointRounding.ToEven` unless you pass a mode. It's usually called banker's rounding: a value exactly halfway between two neighbours goes to the one whose last digit is even.

| Value | ToEven | AwayFromZero |
|---|---|---|
| 0.025 | 0.02 | 0.03 |
| 0.075 | 0.08 | 0.08 |
| 1.125 | 1.12 | 1.13 |
| 1.135 | 1.14 | 1.14 |

With round half up, every midpoint goes up, so over many postings the rounding error builds up in one direction. With half to even, midpoints go up about half the time and down the rest, and the error mostly cancels out.

## The mode is a parameter

Half to even isn't right everywhere. Some products and some tax rules specify half up. So `Multiply` takes the mode as a parameter, with `ToEven` as the default:

```csharp
public Money Multiply(decimal factor, MidpointRounding rounding = MidpointRounding.ToEven) =>
    new(decimal.Round(Amount * factor, Currency.MinorUnits, rounding), Currency);
```

Code that needs another rule has to say it: `Multiply(0.5m, MidpointRounding.AwayFromZero)`. That way the choice shows up in code review.

The number of decimals comes from `Currency.MinorUnits`, not a hard-coded 2. The yen has no minor unit and the Bahraini dinar has three.

## Rejecting amounts that are too precise

Rounding after a multiplication can't be avoided. Input is another matter. If 10.005 EUR arrives, storing it as 10.00 or 10.01 creates half a cent of difference that nobody will be able to trace later. So `Money.Of` refuses it:

```csharp
public static Money Of(decimal amount, Currency currency)
{
    ArgumentNullException.ThrowIfNull(currency);

    // a ledger that silently rounds on the way in loses cents nobody can explain later
    if (decimal.Round(amount, currency.MinorUnits) != amount)
    {
        throw new ArgumentException(
            $"{amount} has more than {currency.MinorUnits} decimal places for {currency}.", nameof(amount));
    }

    return new Money(amount, currency);
}
```

`Money.Of(1.5m, Currency.FromCode("JPY"))` and `Money.Of(0.001m, Currency.Eur)` both throw. Trailing zeros are accepted: `1.2000m` passes, and it compares equal to `1.20m`.

## Other checks in Money

- Adding EUR to USD throws a `CurrencyMismatchException`.
- Comparing amounts in different currencies throws too. Converting needs a rate and a date, and a value object has neither.
- The amount is a `decimal`, because `double` can't represent 0.1 exactly.

The code and tests are in [LedgerCore.Domain/Monetary](https://github.com/softvasco/ledger-core/tree/main/src/LedgerCore.Domain/Monetary).
