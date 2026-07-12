---
title: "Stop Burning Tokens: Five Things That Actually Cut My AI Agent Costs"
date: 2026-03-21 00:00:00 +0600
categories: [AI, Development]
tags: [ai, claude-code, tokens, productivity, developer-tools, context, agents]
description: Grep is the most expensive habit your coding agent has. Here's what actually moved the needle for me — a code graph instead of file scans, a contract file instead of re-explaining, and a skill that makes the model talk like a caveman.
---

Your AI agent's biggest expense is not thinking. It's **reading**.

Ask an agent a question about your codebase and watch what it does. It greps. It gets forty file paths back. It reads six of them, in full, to find the one function that mattered. Those six files might be four thousand lines. You paid for every one of them, and roughly thirty of those lines were relevant.

Then you ask a follow-up, and it does it again.

I've been living in agentic coding tools daily for a while now — a Flutter app, a FastAPI backend, three frontends, a couple of published packages — and my token bill went from "amusing" to "an actual line item." So I went looking for what actually reduced it.

Here's what worked, roughly in order of how much it saved. Some of these are unglamorous. One of them is genuinely stupid and works anyway.

---

## 1. Stop Letting Your Agent Grep. Give It a Graph.

This is the big one, and it's the one I'd tell you to do first.

Grep is a *string* search. Your code is a *graph*. Every time an agent greps, it's flattening a rich structure — functions call functions, classes inherit, tests cover code, modules import modules — into a list of text matches, and then re-deriving the structure by reading whole files.

That's the expensive part. Not the search. The **re-derivation**.

I use [`code-review-graph`](https://github.com/tirth8205/code-review-graph) (by Tirth Kanani — I use it, I didn't build it). It parses your repo with Tree-sitter and builds an actual graph in a local SQLite file, then exposes it to your agent over MCP.

To make this concrete, here's what it holds for one of my repos:

| | |
|---|---|
| **Nodes** | 3,411 — 1,864 functions, 921 classes, 411 files, 215 tests |
| **Edges** | 28,543 — 18,649 `CALLS`, 4,051 `IMPORTS_FROM`, 3,011 `CONTAINS`, 2,249 `TESTED_BY`, 583 `INHERITS` |
| **On disk** | one 45MB `.code-review-graph/graph.db` |

Twenty-eight thousand **typed relationships**. Not "these files contain the string `createSale`" — *this function calls that one, this test covers it, this class inherits from that one.*

So instead of "grep for `createSale`, read four files, infer who calls it," the agent asks the graph:

- `query_graph` with `callers_of` — who calls this?
- `get_impact_radius` — what breaks if I change it?
- `query_graph` with `tests_for` — is this even tested?
- `get_architecture_overview` — what are the subsystems here?

The answers come back as **structured facts**, not as source code the model has to read and reason about. That's the saving. You're not paying to have four thousand lines re-read so the model can rediscover a call graph that could just be looked up.

The project's own benchmarks claim ~8x context reduction — treat vendor numbers with the usual salt, but the direction is right and the mechanism is obviously sound.

Setup is genuinely two commands:

```bash
uvx code-review-graph install     # detects your AI tools, writes MCP config
uvx code-review-graph build       # parse the repo, build the graph
```

It keeps itself fresh through git hooks (incremental re-parse of changed files only — sub-second for a single file), so you don't babysit it.

**The catch:** the model won't use it unless you *tell* it to, emphatically. Left alone, an agent will grep, because grep is what it knows. So the instruction goes in the project's contract file, in language that leaves no room:

> **IMPORTANT: This project has a knowledge graph. ALWAYS use the code-review-graph MCP tools BEFORE using Grep/Glob/Read to explore the codebase.** The graph is faster, cheaper (fewer tokens), and gives you structural context (callers, dependents, test coverage) that file scanning cannot. Fall back to Grep/Glob/Read **only** when the graph doesn't cover what you need.

Which brings me to the next thing.

---

## 2. Write the Contract Once Instead of Re-Explaining It Forever

Every fresh agent session starts with amnesia. And what do you do? You re-explain the project. Every time.

*"We use Postgres, not MySQL. Money is `Decimal`, never float. Don't add offline support, it's deliberate. All strings come from the i18n file, never inline."*

You are paying tokens to say the same paragraph, forever, until you die.

Write it down **once**, in a `CLAUDE.md` at the repo root, and the agent reads it as context instead of you typing it. Not a README — a README explains the project to a *human evaluating* it. This is a **contract**: the rules an agent will otherwise cheerfully violate.

The highest-value lines are always the ones where the globally-good idea is the locally-catastrophic one:

> **Backend owns truth. Clients are online-only for v1** — no offline-first sync. Do not add a local write queue.

Ask any competent model to "make the app work on bad networks" and it will *enthusiastically* build you an offline write queue, because that is a good idea in general. It has no way to know that here, a phone silently replaying a sale is a **wrong ledger**, and a wrong ledger is a lost customer. So you tell it. Once. In a file.

The same file is where the money rules live (`Decimal`, never float), the tenancy rules (every query is shop-scoped or it's a bug), the "we tried this and it broke" rules:

> Forms: React Hook Form + `zodResolver` over **Zod 3**. Do NOT upgrade to Zod 4 until `@hookform/resolvers` ships compatible typings — the admin panel learned this the hard way.

That one sentence prevents an agent from confidently "upgrading" you into a broken build, and then spending twenty thousand tokens debugging it. **The cheapest token is the one spent preventing the mistake.**

---

## 3. The Caveman Skill (Yes, Really)

This one sounds like a joke. It is not a joke and I use it every day.

Roughly half the output of a coding agent is **social**. "Sure! I'd be happy to help you with that. The issue you're experiencing is likely caused by..." — that's forty tokens of throat-clearing before a single fact arrives. Multiply by every response, all day.

The `caveman` skill instructs the model to drop all of it and speak in compressed, telegraphic English. Its own claim is ~75% fewer output tokens with full technical accuracy retained.

```
Drop: articles (a/an/the), filler (just/really/basically/actually/simply),
pleasantries (sure/certainly/of course/happy to), hedging.
Fragments OK. Short synonyms. Technical terms exact. Code blocks unchanged.
Errors quoted exact.

Pattern: [thing] [action] [reason]. [next step].
```

The difference in practice:

> **Normal:** "Sure! I'd be happy to help you with that. The issue you're experiencing is likely caused by an incorrect comparison operator in your authentication middleware, where the token expiry check appears to be using a strict less-than..."
>
> **Caveman:** "Bug in auth middleware. Token expiry check use `<` not `<=`. Fix:"

Same information. Same fix. A fraction of the tokens, and — I did not expect this — **it's faster to read.** I stopped skimming past the preamble to find the answer, because there is no preamble. The answer is the first word.

It has intensity levels. `lite` just kills the filler and keeps normal grammar. `full` drops articles. `ultra` abbreviates aggressively and uses arrows for causality:

> `ultra`: "Inline obj prop → new ref → re-render. `useMemo`."

But the genuinely well-designed part — and the reason I trust it — is that it **knows when to stop**:

> **Auto-Clarity — drop caveman when:**
> - Security warnings
> - Irreversible action confirmations
> - Multi-step sequences where fragment order or omitted conjunctions risk misread
> - Compression itself creates technical ambiguity (e.g. `"migrate table drop column backup first"` — order unclear without articles/conjunctions)

That last example is the whole reason this is safe. "migrate table drop column backup first" — is that *"back up first, then drop"*, or *"drop, then back up"*? One of those loses your data.

**Compression is not free when ambiguity is expensive.** A skill that compresses everything uniformly would eventually save you 75% of the tokens on the message that destroys your database. This one carves out exactly the cases where clarity beats brevity, and that carve-out is what makes the other 95% of the time acceptable.

(It also has classical-Chinese modes — 文言文 — for even denser compression. I have not been brave enough. But I respect it enormously.)

---

## 4. Subagents: Spend Tokens Somewhere Else

Counterintuitive: sometimes the way to spend fewer tokens *in your context* is to spend more of them **somewhere you don't have to pay attention to**.

When I need to answer "how does auth work across these five repos," the naive approach reads twenty files into my main conversation, and every one of those files then sits in context for the **rest of the session**, being re-sent with every subsequent message. That's the killer. A file you read once is a file you pay for over and over.

Delegate it to a subagent instead. The subagent reads the twenty files in *its own* context, and returns a two-paragraph summary. Twenty files' worth of reading, and my context grows by two paragraphs.

The rule I use: **if answering means reading across many files, and I only need the conclusion, delegate it.** If I know exactly which file and which line, just read it — spawning an agent for a one-line lookup is its own kind of waste.

Same instinct as a `SELECT` with a `WHERE` clause. Don't pull the table into the application to filter it there.

---

## 5. Small Things That Add Up

**Point at files, don't make it search.** "Look at `src/deps/idempotency.py`" costs a few tokens. "Find where idempotency is handled" costs a grep, four file reads, and a guess.

**Start a fresh session when the topic changes.** Context is cumulative and you re-send all of it every turn. Debugging a CSS bug in a conversation still carrying 40k tokens of unrelated database schema means you're paying for that schema on every message. Long sessions have gravity. Escape them.

**Let it fail before you explain.** I used to write elaborate preemptive instructions covering every edge case I could imagine. Most of that was wasted — the model would have got it right anyway. Now I let it try, and I only spend tokens explaining the thing it *actually* got wrong. Cheaper, and a better signal about what's genuinely non-obvious about my codebase.

**Then write that down.** The thing it got wrong goes into `CLAUDE.md`, and now it's wrong exactly once, forever, instead of once per session.

---

## The Shape of the Thing

Every item on this list is the same idea wearing a different hat:

**Don't make the model re-derive what could have been looked up.**

- The graph means it doesn't re-derive the call graph from source.
- `CLAUDE.md` means it doesn't re-derive your architecture from your code, or from you.
- Caveman means it doesn't spend tokens performing politeness at you.
- Subagents mean the re-derivation happens in a context you throw away.

Tokens are just the price you pay for a model figuring something out. Every time it figures out the same thing twice, you paid twice.

The graph and the contract file were the two that actually changed my bill. Caveman changed my *day* — the answers just arrive faster now, with no preamble to skim past.

Start with the contract file. It's a text file. It costs nothing. And you have already typed its contents, out loud, into a chat box, more times than you'd like to count.

---

**Tags:** #AI #ClaudeCode #Tokens #Productivity #DeveloperTools #Agents
