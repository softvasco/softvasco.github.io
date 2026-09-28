---
title: "Normalising payee names: where NFKD stops"
description: "Unicode normalisation strips most accents for free. Then you meet ß, ø and ł."
repo: payee-match
---

Since 9 October 2025, every payment provider in the EU has had to check the payee name against the IBAN before a euro credit transfer goes out. That's Verification of Payee. The answer is one of four codes: match, close match, no match, or "can't check". The EPC scheme says how the two banks talk to each other, but for the matching itself it only gives guidelines.

I'm building an open matcher for this, [payee-match](https://github.com/softvasco/payee-match). The first step is boring on purpose: turn both names into something you can compare. "João Conceição" typed by the payer and "JOAO CONCEICAO" in the bank's records should look the same before any fuzzy logic runs.

## The easy 90%

.NET gives you most of it with Unicode normalisation. `NormalizationForm.FormKD` splits a letter with an accent into the base letter plus a separate combining mark. After that you drop anything in the `NonSpacingMark` category and the accents are gone.

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

That one loop handles `ã`, `ç`, `ü`, `é`, `ñ`, and the Czech ones people usually forget, like `ř` and `ť`. "Dvořák Šťastný" comes out as "dvorak stastny".

I picked KD over D on purpose. The K means compatibility, so it also flattens things that only look different: the `ﬁ` ligature becomes `f` and `i`, and full-width `ＡＢＣ` becomes plain `ABC`. You don't see those often in a banking app, but copy and paste from a PDF brings in all sorts.

## The letters NFKD leaves alone

Then the tests with German, Nordic and Polish names start failing.

| Input | After NFKD | What you want |
|---|---|---|
| Straße | Straße | strasse |
| Søren | Søren | soren |
| Łukasz | Łukasz | lukasz |
| Ægir | Ægir | aegir |
| Þórsson | Þorsson | thorsson |

None of these are accented letters. `ß`, `ø`, `ł`, `æ` and `þ` are letters in their own right, so Unicode has no decomposition for them and there's nothing to strip. `ToLowerInvariant` doesn't help either. Lower-casing `ß` gives you `ß`.

So there's a small, explicit table for them:

```csharp
// letters that NFKD leaves alone because they aren't an accent on a base letter
private static readonly FrozenDictionary<char, string> Folds = new Dictionary<char, string>
{
    ['ß'] = "ss", ['æ'] = "ae", ['œ'] = "oe", ['ø'] = "o", ['ł'] = "l",
    ['đ'] = "d", ['ð'] = "d", ['þ'] = "th", ['ı'] = "i", ['ŀ'] = "l",
}.ToFrozenDictionary();
```

It's short, and I like that it's short. Every entry is a decision someone can read and argue with, which matters for a check that has to be explained to a customer, or to an auditor.

## Punctuation is not all the same

The last part is splitting into words, and two characters need special treatment.

Apostrophes join. "O'Neill" and "D'Almeida" are one word, and a payer might type them with a straight quote, a curly one or none at all. Dropping the apostrophe entirely gives `oneill` and `dalmeida` in every case.

Dots split. It's tempting to treat "J.M. Silva" like "O'Neill" and glue it together, but then you get `jm silva` and throw away the fact that those were two initials. A later step matches initials against full given names ("J. M. Silva" against "João Manuel Silva"), so the normaliser keeps them apart: `j m silva`.

Everything else that isn't a letter or a digit ends the current word. Hyphens, commas, `&` and extra spaces all collapse, so "Silva-Santos, Ana" becomes `silva santos ana`.

## What it doesn't do (yet)

This step deliberately doesn't know anything about names. It doesn't drop titles like "Dr", and it doesn't know that "Lda" and "Unipessoal" are Portuguese company forms. That comes next, with a list per country, because "Silva Lda" against "Silva, Unipessoal Lda" is a very different question from "Silva Construções SA" against "Silva Consultores SA".

The normaliser and its tests are [on GitHub](https://github.com/softvasco/payee-match/blob/main/src/PayeeMatch.Core/Names/NameNormaliser.cs). If you have a name from your language that comes out wrong, I'd like to see it.
