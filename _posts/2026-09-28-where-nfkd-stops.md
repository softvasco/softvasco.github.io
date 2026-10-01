---
title: "Normalising payee names: where NFKD stops"
description: "NFKD removes most accents from a name before fuzzy matching. It does nothing for ß, ø or ł, so those need a small table, and dots and apostrophes need their own rules."
repo: payee-match
image: /assets/cards/where-nfkd-stops.png
---

Since 9 October 2025, payment providers in the EU have to check the payee name against the IBAN before they send a euro credit transfer. This is Verification of Payee. The payee's bank answers with one of four codes: match, close match, no match, or check not possible. The EPC rulebook defines the messages between the two banks, but for the name matching it only gives guidelines, so each bank decides how to do it.

I'm writing an open-source matcher for .NET, [payee-match](https://github.com/softvasco/payee-match). Before any fuzzy matching, both names have to be brought to the same form, so that "João Conceição" typed by the payer and "JOAO CONCEICAO" in the bank's records compare as equal. This post is about that step.

## Removing accents

.NET does most of it. `string.Normalize(NormalizationForm.FormKD)` decomposes an accented letter into the base letter followed by a combining mark. Skip everything in the `NonSpacingMark` category and the accents are gone:

```csharp
foreach (var c in name.Normalize(NormalizationForm.FormKD))
{
    if (CharUnicodeInfo.GetUnicodeCategory(c) == UnicodeCategory.NonSpacingMark)
    {
        continue;
    }
    // ...
}
```

That covers ã, ç, ü, é and ñ, and Czech letters like ř and ť too. "Dvořák Šťastný" becomes "dvorak stastny".

I used KD rather than D. The compatibility form also turns the `ﬁ` ligature into `f` and `i`, and full-width `ＡＢＣ` into `ABC`. You won't see either often in a payment form, but a name pasted from a PDF can bring them in.

## Letters with no accent to remove

The first German, Nordic and Polish names in the tests failed:

| Input | After NFKD | Expected |
|---|---|---|
| Straße | Straße | strasse |
| Søren | Søren | soren |
| Łukasz | Łukasz | lukasz |
| Ægir | Ægir | aegir |
| Þórsson | Þorsson | thorsson |

ß, ø, ł, æ and þ are letters of their own, not a base letter with an accent, so Unicode has no decomposition for them. Lower-casing doesn't touch them either: `"ß".ToLowerInvariant()` is still `"ß"`.

They go through a lookup table instead:

```csharp
// letters that NFKD leaves alone because they aren't an accent on a base letter
private static readonly FrozenDictionary<char, string> Folds = new Dictionary<char, string>
{
    ['ß'] = "ss", ['æ'] = "ae", ['œ'] = "oe", ['ø'] = "o", ['ł'] = "l",
    ['đ'] = "d", ['ð'] = "d", ['þ'] = "th", ['ı'] = "i", ['ŀ'] = "l",
}.ToFrozenDictionary();
```

I kept it as an explicit list. A VoP result may have to be explained to a customer or an auditor, and with ten entries anyone can check exactly what was replaced.

## Splitting into words

Next the name is split into tokens. Most punctuation just ends a word: hyphens, commas, `&`, repeated spaces. "Silva-Santos, Ana" becomes `silva santos ana`.

Apostrophes are different. O'Neill and D'Almeida should stay one word, and people type them with a straight quote, a curly one or nothing at all. Dropping the apostrophe gives `oneill` and `dalmeida` in every case.

At first I wanted dots to behave the same way, but then "J.M. Silva" becomes `jm silva` and the two initials are lost. A later step compares initials with full given names ("J. M. Silva" against "João Manuel Silva"), so dots split: `j m silva`.

## Titles and company forms

The normaliser knows nothing about names. After it runs, titles like "Dr" and company forms like "Lda" or "GmbH" are still there. A separate step removes them, and it keeps the legal form to one side instead of throwing it away, because "Silva Lda" and "Silva SA" are two different companies. I'll write about that one when the matcher uses it.

The normaliser and its tests are in [NameNormaliser.cs](https://github.com/softvasco/payee-match/blob/main/src/PayeeMatch.Core/Names/NameNormaliser.cs). If a name in your language comes out wrong, please open an issue with it.
