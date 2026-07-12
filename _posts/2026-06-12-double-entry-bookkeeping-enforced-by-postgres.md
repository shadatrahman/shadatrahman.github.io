---
title: "Debits Must Equal Credits — Letting Postgres Enforce 500 Years of Accounting"
date: 2026-06-12 00:00:00 +0600
categories: [Development, Architecture]
tags: [postgres, accounting, double-entry, fintech, database, triggers, saas]
description: A trial balance that doesn't balance isn't a report — it's a lie with a total at the bottom. So I made an unbalanced journal entry physically impossible to commit, with a DEFERRABLE constraint trigger.
---

Luca Pacioli published the double-entry system in 1494. It has survived the Medici, the Dutch East India Company, the invention of the corporation, two world wars, and Enron.

It is not going to be defeated by my `INSERT` statement.

But it very nearly was — and the thing that saved it was a Postgres feature I'd never had a reason to use before: a **`DEFERRABLE INITIALLY DEFERRED` constraint trigger**.

This is a post about building a real general ledger for a shop-accounting app, and about the difference between *checking* an invariant and making it *impossible to violate*.

---

## Why a Shop in Dhaka Needs a General Ledger

Amar Khata started as a *khata* — a notebook. Shop owners record sales, track who owes them money (বাকি / *baki*), and see a daily summary. Simple operational tables: `sales`, `payments`, `customers.total_due`.

That works right up until the shop owner asks a question the operational tables cannot answer:

> "I know my customers owe me ৳45,000. But **am I actually making money?**"

That's not a question about dues. That's a question about *the whole business* — cash, stock, what she owes suppliers, rent, her own drawings from the till. Answering it requires knowing that the books **balance**: that every taka has both a source and a destination.

Which means a real chart of accounts, real journal entries, and a real **trial balance**.

And a trial balance has exactly one job: **prove that total debits equal total credits.** If it doesn't balance, it isn't a report — it's a lie with a total at the bottom. Worse than no report, because she'd *believe* it.

So the whole feature rests on one invariant:

> **For every journal entry: Σdebit = Σcredit.**

Everything else is presentation. If that holds, the books are sound. If it can ever be violated — even once, even by a bug I fix the next day — every number the app has ever shown becomes suspect.

---

## The Chart of Accounts (in Bangla)

First, the accounts. Standard five types — Asset, Liability, Equity, Income, Expense — seeded for every shop, and named in the language the shop owner actually thinks in:

```python
_DEFAULT_CHART = (
    ("1000", "Cash",                        "নগদ টাকা",              "ASSET",     "CASH",           True),
    ("1100", "Accounts Receivable (Baki)",  "বাকি — গ্রাহক",          "ASSET",     "AR",             True),
    ("1200", "Inventory",                   "পণ্য মজুদ",              "ASSET",     None,             True),
    ("2000", "Accounts Payable",            "দেনা — সরবরাহকারী",     "LIABILITY", "AP",             True),
    ("3000", "Owner's Capital",             "মালিকের মূলধন",         "EQUITY",    None,             True),
    ("3100", "Owner's Drawings",            "মালিকের উত্তোলন",        "EQUITY",    None,             True),
    ("3900", "Opening Balance Equity",      "প্রারম্ভিক উদ্বৃত্ত",      "EQUITY",    "OPENING_EQUITY", True),
    ("4000", "Sales Revenue",               "বিক্রয়",                 "INCOME",    "SALES",          True),
    ("5000", "Purchases / COGS",            "ক্রয়",                   "EXPENSE",   "PURCHASES",      True),
    ("5100", "Rent",                        "দোকান ভাড়া",            "EXPENSE",   None,             True),
    # …
)
```

Note the `system_key` column — `CASH`, `AR`, `AP`, `SALES`. Those are the anchors that let the app *project the operational tables into the ledger* without making the shop owner ever type the word "debit." She records a সেল. The ledger figures out that this means **debit Accounts Receivable, credit Sales Revenue**, and she never has to know.

That's the design goal: a real double-entry ledger underneath, a notebook on top.

---

## Constraint #1: A Line Is a Debit *or* a Credit

Before we get to balancing entries, each individual *line* has to be sane. Two `CHECK` constraints, right in the table definition:

```python
sa.CheckConstraint("debit >= 0 AND credit >= 0", name="journal_lines_non_negative"),
sa.CheckConstraint(
    "(debit > 0 AND credit = 0) OR (credit > 0 AND debit = 0)",
    name="journal_lines_one_sided",
),
```

The first bans negative amounts. **A negative debit is a credit**, and if you allow both spellings of the same thing, you have two ways to represent every transaction, and eventually two parts of your codebase will disagree about which one they're looking at. Pick one. Enforce it.

The second bans a line that's *both* a debit and a credit, and a line that's *neither* (both zero — a ghost line that contributes nothing but pads the entry). A line moves money in exactly one direction. It is never ambiguous, and it is never a no-op.

These are cheap. Postgres evaluates them per row, they cost nothing, and they eliminate an entire category of nonsense before it can reach the interesting logic.

Also, obviously: `Numeric(12, 2)`. Never float. If you are storing money in a float, no trigger I can write will save you.

---

## Constraint #2: The Entry Must Balance — And Here's Where It Gets Hard

Now the real invariant. Σdebit = Σcredit, per entry.

The obvious move is a trigger on `journal_lines`:

```sql
CREATE TRIGGER journal_lines_balanced
    AFTER INSERT OR UPDATE OR DELETE ON journal_lines
    FOR EACH ROW EXECUTE FUNCTION assert_journal_balanced();
```

And this **does not work.** Not "is inefficient" — does not work *at all*. It makes the feature impossible.

Think about what inserting an entry actually looks like. A sale of ৳500 on credit is two rows:

```
INSERT INTO journal_lines (entry_id, account_id, debit,  credit) VALUES (e1, AR,    500.00, 0);
INSERT INTO journal_lines (entry_id, account_id, debit,  credit) VALUES (e1, SALES, 0,      500.00);
```

The trigger fires after the **first** row. At that instant, entry `e1` has debits of 500 and credits of 0.

**It is unbalanced.** Because of course it is — the other half hasn't been written yet. The trigger raises, the transaction dies, and you have built a ledger in which it is impossible to record a transaction. Every journal entry is transiently unbalanced in the middle of being written. That's not a bug in the data; that's what writing more than one row means.

You could dodge it. Insert both lines in a single multi-row `INSERT`, or defer the check to the application layer, or use a statement-level trigger and hope nobody ever writes lines one at a time.

All of those are workarounds for the fact that I asked the wrong question. I was checking the invariant **after each row**, when the invariant is only *meaningful* **at the end of the transaction**.

Postgres has exactly the thing for this, and I'd never needed it before.

---

## `DEFERRABLE INITIALLY DEFERRED`

```sql
CREATE CONSTRAINT TRIGGER journal_lines_balanced
    AFTER INSERT OR UPDATE OR DELETE ON journal_lines
    DEFERRABLE INITIALLY DEFERRED          -- ← the entire post is this line
    FOR EACH ROW EXECUTE FUNCTION assert_journal_balanced();
```

A **`CONSTRAINT TRIGGER`** (not a plain trigger) can be deferred. `INITIALLY DEFERRED` means: don't run at statement time — **queue me up and run me at `COMMIT`.**

So now the sequence is:

1. `INSERT` line one (AR, debit 500). Trigger queued, not fired.
2. `INSERT` line two (SALES, credit 500). Trigger queued, not fired.
3. `COMMIT` → **now** the trigger runs. Debits 500, credits 500. Balanced. Commit succeeds.

And if the second line never comes — if the code crashed, if a bug wrote only one leg, if someone opened `psql` and inserted a single row by hand and typed `COMMIT` — the trigger fires at commit time, finds 500 ≠ 0, and **kills the transaction.**

The invariant now holds at exactly the boundary where it is meaningful: the transaction. Inside the transaction you may be transiently unbalanced, because you're mid-thought. At the moment your work becomes *visible to anyone else*, it balances. Or it does not exist.

**There is no third outcome.** An unbalanced journal entry cannot be committed to this database. Not by my API, not by a background job, not by a future teammate, not by an AI agent writing a migration at 2am, not by me in a psql shell with a coffee and bad judgment.

Here's the function it runs:

```sql
CREATE OR REPLACE FUNCTION assert_journal_balanced() RETURNS trigger
LANGUAGE plpgsql AS $$
DECLARE
    eid uuid; d numeric; c numeric;
BEGIN
    eid := COALESCE(NEW.entry_id, OLD.entry_id);

    -- Entry removed (cascade delete) → nothing left to balance.
    IF NOT EXISTS (SELECT 1 FROM journal_entries WHERE id = eid) THEN
        RETURN NULL;
    END IF;

    SELECT COALESCE(SUM(debit), 0), COALESCE(SUM(credit), 0)
      INTO d, c FROM journal_lines WHERE entry_id = eid;

    IF d <> c THEN
        RAISE EXCEPTION
            'journal entry % is unbalanced (debit=%, credit=%)', eid, d, c
            USING ERRCODE = 'check_violation';
    END IF;
    RETURN NULL;
END;
$$;
```

### The cascade-delete trap

That `IF NOT EXISTS (SELECT 1 FROM journal_entries WHERE id = eid) RETURN NULL` guard is not decoration. It's the second bug this design hands you, and it's a good one.

Delete a journal entry. Its lines cascade-delete. The trigger — which fires on `DELETE` too — runs at commit for each removed line and asks "does entry `e1` still balance?"

Entry `e1` doesn't *exist* anymore. Its lines are gone. `SUM(debit)` over zero rows is `NULL`, coalesced to 0; same for credit. So 0 = 0, and you'd think it passes by luck.

But that's luck, not logic, and luck runs out. Depending on the order the cascade removes rows, you can absolutely observe a half-deleted entry — some lines gone, some still there — and *that* genuinely doesn't balance. Your delete fails with a bizarre "unbalanced entry" error about an entry you were deliberately destroying.

So: if the parent entry is gone, there's nothing to balance. Return and let it go. **The invariant is about entries that exist.**

### `COALESCE(NEW.entry_id, OLD.entry_id)`

Small, but worth naming. On `INSERT`/`UPDATE` you get `NEW`. On `DELETE` you only get `OLD` — `NEW` is `NULL`. One trigger function serving all three events has to reach for whichever one it's been handed.

---

## Defense in Depth: Three Layers, Deliberately

The database trigger is the *last* line of defense, not the only one:

**Layer 1 — the service.** `create_journal_entry()` validates before it writes: the accounts exist, are active, belong to this shop, and the lines balance. This is where a user gets a *useful error message* in Bangla, not a Postgres exception.

```python
async def create_journal_entry(session, *, shop_id, user_id, entry_date, memo, lines, source=MANUAL):
    """Validate + persist a balanced journal entry. Caller owns the transaction."""
    accounts = await _load_active_accounts(session, shop_id=shop_id, account_ids={...})
    _validate_lines(lines, accounts)
    ...
```

**Layer 2 — the `CHECK` constraints.** Non-negative, one-sided. Per row, free.

**Layer 3 — the deferred constraint trigger.** At commit. Unbypassable.

If layer 1 is doing its job, layer 3 should *never* fire in production. That is exactly the point. Layer 3 isn't there for the code path I thought about — it's there for the one I didn't. It's there for the migration script, the data backfill, the admin tool, the `psql` session. It's there for whatever I write next year having forgotten I wrote this.

There's a test named for precisely this, and I'm fond of it:

```
test_db_trigger_is_last_defense
```

It bypasses the service layer entirely, writes an unbalanced entry straight at the database, and asserts that Postgres refuses it. Because a defense you've never actually fired is a defense you're merely *hoping* you have.

(The ledger tables also sit under the same [Row-Level Security](/posts/postgres-rls-multi-tenancy-from-migration-one/) as every other business table, so one shop's chart of accounts is invisible to every other shop. Tenant isolation and accounting integrity are separate invariants, and both are enforced by the database.)

---

## Projecting the Notebook Into the Ledger

Here's the part that makes this usable by someone who has never heard the word "debit."

The shop owner does not create journal entries. She records sales and payments, exactly as before. The ledger **projects** those operational tables into subledgers using the `system_key` anchors:

- a BAKI (credit) sale → debit **AR**, credit **SALES**
- a customer payment → debit **CASH**, credit **AR**
- a supplier payable → debit **PURCHASES**, credit **AP**
- a payment to a supplier → debit **AP**, credit **CASH**

Those projections are computed at read time from tables that already exist. What gets *stored* as real journal entries are only the things that have nowhere else to live: **manual entries** (rent, salaries, the owner taking cash out of the till) and **opening balances**.

Opening balances are the fun one. A shop that's been running for eleven years doesn't start from zero — it starts with cash in the drawer, stock on the shelves, and money owed in both directions. Those figures come from the real world and they will not balance on their own. So they're plugged to **Opening Balance Equity** (`3900`, প্রারম্ভিক উদ্বৃত্ত): whatever it takes to make the opening entry balance goes there. That's not a fudge — it's a real accounting practice, and it's what that account is *for*. The shop's history, whatever it was, lands in equity, and from that moment forward every taka is tracked.

The trial balance then sums the projected subledgers and the stored entries, as of any date the owner picks, with day boundaries computed in `Asia/Dhaka` — because a shop's day ends when the shop closes, not at midnight UTC.

And then the whole thing balances. It has to. There is no code path in which it doesn't.

---

## What I'd Tell You

**1. Ask *when* your invariant is true, not just *what* it is.** "Debits equal credits" is false for most of the lifetime of the transaction that creates the entry, and that isn't a defect — it's what multi-row writes mean. I lost an afternoon to a trigger that was *correct* and fired at the *wrong moment*. `DEFERRABLE INITIALLY DEFERRED` moves the check to the only boundary where the invariant is meaningful: the commit.

**2. `CONSTRAINT TRIGGER` is not the same as `TRIGGER`.** Only a constraint trigger can be deferred. If you've been reaching for statement-level triggers to dodge the mid-transaction problem, this is the tool you actually wanted.

**3. Handle the cascade delete.** Any trigger that validates an aggregate across child rows will fire during a parent's cascade delete and see a half-demolished set. Check that the parent still exists, and bail if it doesn't.

**4. The database is the last line, not the only line.** Validate in the service so users get real error messages. Enforce in the database so that *nothing* — no script, no migration, no agent, no future you — can get around it. Both. The one at the bottom is the one you're betting the product on.

**5. Test the last line by actually firing it.** `test_db_trigger_is_last_defense` bypasses every application safeguard and shoots straight at Postgres. An unfired defense is a hypothesis.

**6. Never negative amounts.** A negative debit is a credit. Two representations of one fact is a bug generator with a long fuse.

---

## The Point

I could have written a service-layer check that sums the lines and raises if they don't match, shipped it, and been fine. Probably. For a while.

But "probably fine, for a while" is not a thing you get to say about someone's books. The shop owner is going to look at that trial balance and make decisions about her life with it — whether she can afford stock, whether she can extend more credit, whether the business is worth continuing.

The number at the bottom has to be **true**. Not usually true. Not true unless there's a bug.

Postgres will enforce that for free, at commit, forever, against code I haven't written yet. All I had to do was ask it at the right moment.

Pacioli would have loved `DEFERRABLE INITIALLY DEFERRED`.

---

**Tags:** #Postgres #Accounting #DoubleEntry #Fintech #Database #Triggers
