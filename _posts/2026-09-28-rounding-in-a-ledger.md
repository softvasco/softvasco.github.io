---
title: "Why 0.05 × 0.5 is 0.02 in my ledger"
description: "Banker's rounding, rejecting fractions of a cent, and making the rounding rule a parameter instead of an accident."
repo: ledger-core
---

Half of five cents is two and a half cents. A ledger can't store half a cent, so something has to give. In [ledger-core](https://github.com/softvasco/ledger-core) the answer is two cents, and the reason is worth a short post.

## Midpoints go to the even number

.NET's `decimal.Round` uses `MidpointRounding.ToEven` unless you tell it otherwise. It's also called banker's rounding. When a value sits exactly halfway, it goes to whichever neighbour ends in an even digit:

| Value | ToEven | AwayFromZero |
|---|---|---|
| 0.025 | 0.02 | 0.03 |
| 0.075 | 0.08 | 0.08 |
| 1.125 | 1.12 | 1.13 |
| 1.135 | 1.14 | 1.14 |

The school rule, round half up, always pushes midpoints the same way. Across millions of postings that turns into a small but steady drift in one direction. Half to even pushes up about as often as down, so on average the rounding cancels itself out.

## But it's not always the right rule

Some products and some tax rules want half up, and that's a business decision, not something the Money type should decide by accident. So the rule is a parameter with a sensible default:

```csharp
public Money Multiply(decimal factor, MidpointRounding rounding = MidpointRounding.ToEven) =>
    new(decimal.Round(Amount * factor, Currency.MinorUnits, rounding), Currency);
```

The call site says `Multiply(0.5m)` when the default is fine and `Multiply(0.5m, MidpointRounding.AwayFromZero)` when a product needs something else. When someone reads that line in two years, the choice is right there.

Note the `Currency.MinorUnits`. Rounding to two places is wrong for yen (zero decimals) and for Bahraini dinar (three). The currency knows how precise it is, so the rounding asks it.

## Don't round on the way in

Rounding after arithmetic is unavoidable. Rounding input is a different story. If something hands the ledger 10.005 EUR, the worst option is to quietly store 10.00 or 10.01. Now there's a cent somewhere that nobody can explain.

So creating money that's finer than its currency allows fails straight away:

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

`Money.Of(1.5m, Currency.FromCode("JPY"))` throws. So does `Money.Of(0.001m, Currency.Eur)`. Trailing zeros are fine though: `1.2000m` is still exactly 1.20, and `decimal` equality agrees.

## The small stuff around it

A few other rules sit in the same type, and each one is a bug I'd rather not meet in production:

- Adding EUR to USD throws a `CurrencyMismatchException` instead of producing a number.
- Comparing amounts in different currencies throws too. "Is 10 EUR more than 10 USD?" isn't a question a value object should answer.
- Amounts are always `decimal`. `double` can't represent 0.1 exactly, and in a ledger that gets old fast.

None of this is clever. It's the kind of thing that's cheap to get right on day one and expensive to fix once real postings depend on it. The code and its tests are in [ledger-core](https://github.com/softvasco/ledger-core/tree/main/src/LedgerCore.Domain/Monetary).
