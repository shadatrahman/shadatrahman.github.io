---
title: "The Invisible Characters That Broke Every Quiz"
date: 2026-06-28 00:00:00 +0600
categories: [Development, AI]
tags: [go, regex, unicode, llm, rag, debugging, ai, text-processing]
description: Every YouTube quiz failed on staging. Every pasted-text quiz passed. The bug was a marker our own pipeline inserted, a validator that couldn't see through it, and Go's \s quietly refusing to match a non-breaking space.
---

Every YouTube quiz generation on staging failed. Every pasted-text one passed.

Not *most*. Not *usually*. **Every** one, on one side; **zero** failures on the other. Five YouTube jobs, five failures. Eight pasted-text jobs, eight passes.

When a bug is that clean, it's a gift. A flaky bug is a haunting. A bug with a perfect correlation is a **fact**, and facts have causes you can find.

The cause turned out to be a character you cannot see, inserted by our own code, that a regex could not match — and it took down the entire feature.

---

## What the Feature Does

The product turns a source — a PDF, a YouTube video, pasted text — into a quiz. Upload a lecture, get ten questions.

The dangerous part of that sentence is **"get ten questions."** An LLM asked to write quiz questions about a document will happily write questions about things the document does not say. Confidently. In excellent prose. That is the entire problem with the technology, and if you're building on it, hallucination isn't an edge case you handle later — it's the thing you're actually engineering against.

So every generated question has to carry a **verbatim quote** from the source, and that quote is a **hard validation gate**:

```go
} else if !quoteFound(req.SourceText, q.Quote) {
    p = append(p, label+": the quote was not found in the source; "+
                  "quote a short phrase verbatim from the source text")
}
```

If the model claims the source says something, it must show us **where**. We then find that quote in the source text and resolve it to a citation — a page number for a PDF, a timestamp for a video — so the user can jump straight to it and check.

Quote not found → the question is rejected → the problems go back to the model for **one** repair attempt → and if the repair still fails, the **whole generation dies**:

```
generate: could not produce a valid quiz after 1 repair attempt(s)
```

There's no partial success. One bad question kills the batch. Remember that — it's why a 15% problem became a 100% outage.

---

## The Markers We Inserted Ourselves

To resolve a quote to a *page* or a *timestamp*, the extractor has to leave breadcrumbs in the text.

The PDF extractor writes a marker at each page boundary:

```go
fmt.Fprintf(&b, "[page %d]\n%s\n\n", i, t)
```

And the YouTube path transcribes to SRT, then prefixes **each transcript cue** with its timestamp:

```
[0:00] the mitochondrion is the powerhouse
[0:04] of the cell, and in this lecture we
[0:09] will look at how ATP synthesis
```

So the "source text" the model sees is not clean prose. It is prose **shot through with markers that we put there**.

And those markers are, from the reader's point of view, **invisible**. When the model quotes a sentence that happens to span a cue boundary, it quotes what a human would read:

> "the mitochondrion is the powerhouse of the cell"

That is a faithful, verbatim, honest quote. The model did nothing wrong.

But the string in our source text is:

> `the mitochondrion is the powerhouse [0:04] of the cell`

The model's quote is **not** a substring. Never was. Couldn't be.

Our validator said: *quote not found in source.* Which is to say — our validator accused the model of hallucinating, because of a marker **we** had inserted.

---

## Why YouTube and Not PDFs

Here's the beautiful part, and the reason the correlation was so clean.

It's a matter of **marker density**.

- **Pasted text** has no markers at all. A quote can never cross one. **Zero failures, structurally.**
- **PDFs** get a `[page N]` marker roughly every 3,000 characters. A twelve-word quote crossing a page break is possible, but rare.
- **YouTube transcripts** get a `[m:ss]` marker **once per cue** — and on the real staging transcripts, cues averaged about **300 characters**.

A model quoting a natural ~12-word span, in text with a marker every 300 characters, is going to cross a marker *constantly*.

We measured it on the actual failing transcripts:

> **70 of 468** twelve-word spans were rejected, and **every single rejection was a marker-crossing span.**

About **15%** of faithful quotes rejected. Which sounds survivable, until you remember the repair loop: 15% of quotes failing means **2–3 bad questions per 10-question quiz** — and the observed failures were exactly that. `3/10`. `2/10`. `5/20`.

And since a single unrepairable question kills the entire generation, a 15% per-quote failure rate became a **100% per-job failure rate**.

That's the whole shape of the bug. A moderate, well-defined statistical problem at the *quote* level, amplified into total, deterministic failure at the *job* level by an all-or-nothing repair loop.

---

## Two Functions That Had Quietly Stopped Agreeing

Digging in, there was a second bug underneath the first — the kind you only find because you went looking for the first one.

**Two different pieces of code answered the question "is this quote in the source?" and they used different logic.**

The validation gate used `quoteFound`, which fell back to a lossy collapse-and-contain check:

```go
func quoteFound(source, quote string) bool {
    q := strings.TrimSpace(quote)
    if q == "" {
        return false
    }
    if indexFold(source, q) >= 0 {
        return true
    }
    return strings.Contains(collapseWS(source), collapseWS(q))   // ← lossy
}
```

The *locator* — the thing that resolves a quote to a page or timestamp — used `indexFoldWS`, a tolerant regex match added in an earlier fix.

So the gate and the locator had **drifted apart**. They could disagree about whether a quote was present. One of them would say "found it, this quote is legitimate," and the other would say "can't locate it," and the citation would silently degrade.

Neither had any concept of markers.

The fix routes **both** through the same function:

```go
func quoteFound(source, quote string) bool {
    q := strings.TrimSpace(quote)
    if q == "" {
        return false
    }
    if indexFold(source, q) >= 0 {
        return true
    }
    start, _ := indexFoldWS(source, q)     // ← same path the locator uses
    return start >= 0
}
```

`collapseWS` was deleted outright.

**If two parts of your system answer the same question, they must be the same code.** Not "equivalent" code. Not code that was equivalent when you wrote it. The same function. Otherwise they *will* drift, and the drift will be invisible until it isn't.

---

## And Then Go's `\s` Betrayed Us

The tolerant matcher works by splitting the quote into words and joining them with a separator pattern — "these words, in this order, with *something* between them."

The original separator was the obvious one:

```go
re, err := regexp.Compile(`(?i)` + strings.Join(parts, `\s+`))
```

Whitespace between words. Reasonable. And now we need to teach it that markers are also allowed between words — that `[0:04]` is a thing that can sit between "powerhouse" and "of" without meaning anything.

So the naive fix is `(?:\s|\[page \d+\]|\[\d+:\d{2}\])+` and you go home.

**That fix would have introduced a new bug**, and it's the kind that would have taken weeks to find, because it only shows up on some documents.

Go's `regexp` implements `\s` as **ASCII-only**. It is precisely `[\t\n\f\r ]`.

It does **not** match a non-breaking space (`U+00A0`). It does not match an em-space (`U+2003`). It does not match NEL (`U+0085`). It does not match the vertical tab. It does not match any of the two dozen characters in Unicode's `Zs` category.

And PDF-extracted text is **full of non-breaking spaces**. That's not an exotic edge case — it is what PDF text extraction *produces*, routinely, because PDFs use them for typesetting.

The code being replaced, `collapseWS`, split on `unicode.IsSpace` — which **does** understand all of that. So swapping a `unicode.IsSpace`-based check for a `\s`-based regex would have silently *narrowed* what counted as whitespace, and quietly started rejecting quotes from any PDF that used a non-breaking space.

We'd have fixed the YouTube path and broken the PDF path, and the symptom would have been identical: *"quote not found in source."*

So the separator spells the whitespace class out **by hand**:

```go
// sepPattern matches what may sit *between* two adjacent source words: a run of
// whitespace and/or the extractors' own markers.
//
// The whitespace class is spelled out rather than using \s because Go's regexp \s
// is ASCII-only ([\t\n\f\r ]) while the extracted text can carry the Unicode spaces
// (NBSP, U+0085, Zs) that PDFs are full of.
const sepPattern = `(?:[\t\n\v\f\r \x{0085}\x{00A0}\p{Zs}]|\[page \d+\]|\[\d+:\d{2}\])+`
```

Read that character class slowly:

- `\t\n\v\f\r ` — ASCII whitespace, **plus the vertical tab** that Go's `\s` also omits
- `\x{0085}` — NEL, the Unicode next-line character
- `\x{00A0}` — the non-breaking space, the actual villain
- `\p{Zs}` — and then the entire Unicode "space separator" category, to catch the em-spaces, thin spaces, and ideographic spaces we haven't met yet
- `\[page \d+\]` and `\[\d+:\d{2}\]` — our own markers, treated as *nothing at all*

All wrapped in `(?:...)+` so any run of mixed whitespace and markers between two words collapses to a single separator.

With a regression test that would have caught the whole thing:

```go
src := "the mitochondrion is the powerhouse"   // NBSP, em-space
if !quoteFound(src, "the mitochondrion is the powerhouse") {
    t.Fatal(...)
}
```

---

## Loosening the Gate Without Making It Gullible

Now the paranoid question, and it's the right one to ask: **we just made the hallucination check more permissive. Did we break it?**

The gate exists to catch a model inventing content. If we relax matching too far, we start accepting fabrications, and the guardrail becomes theatre — worse than no guardrail, because now we *trust* it.

The relaxation is deliberately narrow. The words must still appear **consecutively**, in **order**, separated **only** by whitespace and/or our own markers. There is no fuzzy matching, no edit distance, no "close enough." A marker becomes invisible; nothing else does.

And it was tested adversarially, by generating fabrications and confirming they still bounce:

> **0 of 2000** shuffled-word fabrications were accepted, on either transcript.

Two thousand attempts to sneak a fake quote past the loosened gate. Zero got through.

The results on the real data:

> **70 → 0** rejections on the 2.7k-character transcript.
> **613/613** marker-crossing spans matched on the 17.8k-character one.

When a quote does span a marker, it resolves to the page or cue it **starts** in. Which is the honest answer — it's where the sentence begins.

---

## The Bug We Found by Reading

One more thing, and it's the part I'd most want you to take away.

The PDF path has the **same defect**. A sentence crossing a page break is exactly the same failure as a sentence crossing a cue boundary — just rarer, because `[page N]` markers are ~10x sparser than `[m:ss]` ones.

We did not find that by observing it. **No PDF generation job had ever run on staging.** We found it by reading the code and realising the bug class wasn't specific to transcripts, and the same fix covers both.

It's written down in the commit, in plain language, so nobody mistakes it for something that was verified:

> Note this fixes the same defect class on the PDF side (a sentence crossing a page break), found by reading — not by observing. No PDF generation job has ever run on staging, so that path remains unexercised.

That's the sentence I'm proudest of in the whole fix. **Say which of your fixes you have actually seen work and which you have merely reasoned about.** They are not the same thing, they do not deserve the same confidence, and the next person to touch that code — very possibly you, in October — needs to know which is which.

---

## What I'd Tell You

**1. A perfect correlation is a gift.** "All YouTube fails, all pasted-text passes" isn't a symptom, it's a *pointer*. Find the thing that is true of one input class and false of the other, and you have found your bug. Here, that was: markers exist, and how densely.

**2. You are polluting your own inputs.** We inserted `[page N]` and `[m:ss]` into the text ourselves and then treated that text as pristine when validating against it. Any pipeline that annotates a document and then searches that document must decide, *explicitly*, what its annotations mean to the search. Ours meant "nothing" — and we had never said so.

**3. `\s` is a lie in most regex engines.** In Go it is exactly `[\t\n\f\r ]`. It is ASCII. Your text is not. PDF extraction produces non-breaking spaces the way a photocopier produces toner smudges. If you are matching whitespace in text that came from the real world, spell your character class out, and include `\p{Zs}`.

**4. One question, one function.** `quoteFound` and `resolveReference` both answered "is this quote in the source?" with different code, and they had silently drifted. Two implementations of one predicate is a bug with a fuse in it.

**5. Loosening a guardrail obliges you to attack it.** We relaxed the hallucination gate, so we threw 2,000 fabrications at it and proved 0 got through. If you can't state a number like that, you don't know whether you loosened the gate or removed it.

**6. All-or-nothing retry turns a percentage into an outage.** 15% of quotes failing became 100% of jobs failing, purely because one unrepairable question killed the batch. Look hard at your amplifiers — the bug wasn't 100% bad, but the *pipeline* made it 100% fatal.

---

## The Point

The model was doing its job perfectly. It read the transcript, it quoted it faithfully, and we told it that it was lying.

The bug wasn't in the AI. The bug was in a character we inserted, a regex that couldn't see it, and a `\s` that quietly meant something narrower than we assumed.

You cannot see any of those characters. That's rather the point of them.

---

**Tags:** #Go #Regex #Unicode #LLM #RAG #Debugging #TextProcessing
