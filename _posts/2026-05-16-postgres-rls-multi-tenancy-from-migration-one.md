---
title: "If a Query Can Run Without a Tenant Filter, It's a Bug — Postgres RLS from Migration 1"
date: 2026-05-16 00:00:00 +0600
categories: [Development, Architecture]
tags: [postgres, rls, multi-tenancy, security, fastapi, saas, database]
description: Every multi-tenant app is one forgotten WHERE clause away from showing Shop A the customer list of Shop B. So I stopped trusting myself to remember, and made the database refuse.
---

Here's the nightmare, and it's a short one.

I'm building a SaaS for Bangladeshi shop owners. It stores the thing they care about most in the world: who owes them money. Customer names, phone numbers, outstanding dues (বাকি / *baki*), every sale of every day.

Now imagine one shop owner opens the app and sees **another shop's** customer list.

That's not a bug report. That's the end of the product. There is no apology email that recovers from "our software showed your competitor your debtors." In a market built on trust and word-of-mouth, you'd be finished by lunchtime.

So the question that shaped the entire backend was: *how do I make that impossible?* Not unlikely. Not well-tested. **Impossible.**

---

## The Way Everyone Does It (And Why It Terrifies Me)

The standard approach to multi-tenancy is a `WHERE` clause:

```python
async def list_customers(session, shop_id: UUID):
    return await session.execute(
        select(Customer).where(Customer.shop_id == shop_id)   # ← the whole defense
    )
```

Every query filters by tenant. Simple, obvious, works.

And it is held together entirely by **you remembering to type it, every single time, forever.**

Think about what has to stay true for that to hold. Every query in every repository. Every ad-hoc admin script. Every `JOIN` — including the ones where you filtered the parent table but not the child. Every new endpoint written at 1am. Every endpoint written by an AI agent that has never heard of your tenancy model. Every future teammate. Every refactor that moves a query somewhere new and drops a clause on the way.

Miss it **once**, in one query, ever, and you have a cross-tenant data breach.

That's not a security model. That's a streak. And streaks end.

I wanted the database to be the one enforcing this — because the database doesn't get tired, doesn't skip code review, and doesn't have a deadline.

---

## Postgres Row-Level Security, and the Two Words That Actually Matter

Postgres has **Row-Level Security**: policies attached to a table that filter which rows a role can see and write, evaluated by the database on every single query, no matter who wrote it or how.

The rule I wrote into the project's contract on day one:

> **Every business row is tenant-scoped.** Postgres RLS is the primary defense from migration 1; the repository's `shop_id` argument is defense-in-depth, never the sole gate. **If a query can run without a `shop_id` filter, it's a bug.**

Note the ordering. The `WHERE` clause still exists — but it's the *second* line of defense now, not the only one. RLS is what makes the guarantee.

Here's what that actually looks like. Migration `20260505_0100`, the very first migration in the project, before there is a single table:

```python
def upgrade() -> None:
    op.execute("""
        CREATE ROLE app_user LOGIN
            NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT NOREPLICATION
            NOBYPASSRLS;              -- ← this one. right here.
    """)
```

**`NOBYPASSRLS`.** A Postgres role with `BYPASSRLS` walks straight through every policy you write like they're not there. If your application connects as a superuser — and a shocking number of apps do, because that's what the default connection string in the tutorial said — then your beautiful RLS policies are decoration. They are doing *nothing*.

The app connects as `app_user`. `app_user` is not a superuser, cannot bypass RLS, and has had `ALL` privileges revoked on the public schema so it only gets what's explicitly granted back.

Then, migration `20260505_0200` — still before any real feature exists — turns it on:

```python
def _enable_rls(table: str) -> None:
    op.execute(f"ALTER TABLE {table} ENABLE ROW LEVEL SECURITY")
    op.execute(f"ALTER TABLE {table} FORCE ROW LEVEL SECURITY")   # ← and this one
```

**`FORCE ROW LEVEL SECURITY`** is the second footgun, and it's nastier than the first because everything *looks* fine without it.

`ENABLE ROW LEVEL SECURITY` turns policies on — **except for the table's owner.** The owner is exempt by default. And guess which role runs your migrations and therefore owns every table? Right. So in a lot of setups the migration user owns the tables, the app connects as that same user for convenience, and RLS silently applies to nobody at all. Your policies are live, your tests pass, and every row is visible to everyone.

`FORCE` removes the owner exemption. Two words, `ENABLE` and `FORCE`, and you need both.

---

## The Policy, and the Cleverest Line in the Codebase

Here's the actual tenant isolation policy, applied to every business table — `customers`, `sales`, `payments`, `sms_messages`, `daily_summary`, `subscriptions`, `idempotency_keys`:

```sql
CREATE POLICY tenant_isolation ON customers
    AS PERMISSIVE
    FOR ALL
    TO app_user
    USING (
        shop_id = nullif(current_setting('app.current_shop_id', true), '')::uuid
    )
    WITH CHECK (
        shop_id = nullif(current_setting('app.current_shop_id', true), '')::uuid
    );
```

Three things are load-bearing here, and one of them is genuinely subtle.

**`USING` governs reads. `WITH CHECK` governs writes.** `USING` filters which rows you can see (`SELECT`, `UPDATE`, `DELETE`). `WITH CHECK` validates rows you're trying to create. Without `WITH CHECK`, a shop could happily *insert* a row tagged with someone else's `shop_id` — they'd never be able to read it back, but they'd have written into another tenant's data. You need both.

**`FOR ALL`** means this covers `SELECT`, `INSERT`, `UPDATE`, and `DELETE`. Not just reads.

And then there's this:

```sql
nullif(current_setting('app.current_shop_id', true), '')::uuid
```

That `nullif(..., '')` is the most important thing in this post.

`current_setting('app.current_shop_id', true)` reads a session variable. That second argument, `true`, is `missing_ok` — it says "if this setting was never set, don't raise an error, just return empty."

So: what happens if a query runs and **nobody set the tenant scope**? Some middleware didn't run. Some background job connected directly. Some developer opened a psql shell as `app_user` and started poking around.

Without the `nullif`, `current_setting` returns `''`, and `''::uuid` throws a **cast error**. Your query explodes with `invalid input syntax for type uuid: ""`. That's... actually not the worst outcome? It fails. But it fails *loudly and confusingly*, and — this is the part that matters — a developer staring at that error at 2am has a strong incentive to make it go away, and the fastest way to make it go away is to weaken the policy.

With the `nullif`, `''` becomes `NULL`. And in SQL, `shop_id = NULL` is not true. It's not false either — it's `NULL`, which is not `true`, which means **the policy does not pass, and you get zero rows.**

**No tenant scope set → you see nothing.** Not an error. Not everything. *Nothing.*

That's the whole philosophy in one line: **fail closed.** The default state of the system, the state it lands in when something goes wrong or someone forgets a step, is *no access*. You have to actively and correctly prove who you are to see a single row.

Compare that to the `WHERE shop_id = :shop_id` model, whose failure mode — forgetting the clause — is **fail open**, returning every row in the table to whoever asked.

That's the entire difference. Same feature, opposite failure mode.

---

## Setting the Scope Without Poisoning Your Connection Pool

RLS reads `app.current_shop_id`. Something has to set it. And this is where a lot of RLS implementations quietly introduce a *worse* bug than the one they were fixing.

The naive move is `SET app.current_shop_id = '...'` — a **session**-level setting. And that works beautifully right up until you put a connection pooler in front of it.

Because connections get **reused**. Shop A's request finishes, its connection goes back to the pool with `app.current_shop_id` still set to Shop A. Shop B's request grabs that connection. If anything at all goes wrong on Shop B's path — the middleware throws before setting the scope, a retry takes a different code path, an exception unwinds early — Shop B is now executing queries **as Shop A**.

You'd have built a cross-tenant leak *inside your cross-tenant protection*.

The fix is one boolean:

```python
async def apply_tenant_scope(session: AsyncSession, shop_id: UUID) -> None:
    """SQL: SELECT set_config('app.current_shop_id', :id, true).
    `true` = transaction-local (mirrors SET LOCAL)."""
    await session.execute(
        text("SELECT set_config('app.current_shop_id', :sid, true)"),
        {"sid": str(shop_id)},
    )
```

That third argument to `set_config` — `true` — means **transaction-local**. It's the functional equivalent of `SET LOCAL`. When the transaction commits or rolls back, the setting evaporates. It cannot outlive its transaction, so it cannot ride a pooled connection into somebody else's request. Ever.

Then it's wrapped in a FastAPI dependency that every tenant-scoped route uses:

```python
async def get_tenant_session(
    shop_id: Annotated[UUID, Depends(get_current_shop_id)],
    session: Annotated[AsyncSession, Depends(get_session)],
) -> AsyncIterator[AsyncSession]:
    """Yield a session with app.current_shop_id already set.
    Uses an explicit begin() so the SET LOCAL persists for the request."""
    async with session.begin():                    # ← the transaction the scope lives in
        await apply_tenant_scope(session, shop_id)
        yield session
```

Note the explicit `session.begin()`. Transaction-local settings need a transaction to be local *to*. If you're in autocommit mode, every statement is its own transaction, and your carefully-set scope vanishes before the next query runs. The explicit `begin()` is what gives the setting something to live inside for the duration of the request.

And if a route somehow reaches the database *without* an authenticated shop, we don't quietly proceed:

```python
if shop_id is None:
    raise errors.AppError(
        errors.TENANT_SCOPE_MISSING,
        message=(
            "Request reached a tenant-scoped route without an authenticated "
            "shop. Auth middleware did not run or the JWT is missing."
        ),
    )
```

Belt, braces, and a second pair of braces. Even if this check were removed, RLS would still return zero rows — because `nullif` — but I'd rather find out from an explicit error than from a confused user.

---

## The Bootstrap Problem Nobody Warns You About

Here's the one that actually bit me, and I've never seen it mentioned in an RLS tutorial.

**How do you log in?**

Follow the logic. RLS scopes every row to `app.current_shop_id`. That value comes from the JWT. The JWT comes from logging in. Logging in means: user sends a phone number, you look up that user, you send them an OTP, they verify it, you issue a JWT.

But *looking up the user by phone number* is a query against the `users` table. Which is RLS-protected. Which requires `app.current_shop_id` to be set. Which you can't know **because they haven't logged in yet.**

Your authentication system cannot authenticate anybody. The front door is locked from the inside.

The fix is a deliberate, tightly-scoped escape hatch — a *separate* session dependency, used by exactly three endpoints (`request OTP`, `verify OTP`, `refresh token`) and nothing else:

```python
async def auth_bootstrap_session(...):
    """Pre-tenant auth session for the public /auth/* handshake.

    OTP request, OTP verify, and refresh run before any shop scope can be
    derived from a JWT, so the app_user engine + users RLS would hide the
    very rows we need to look up by phone. This dep uses the app_admin
    engine with app.is_superadmin='on' so the lookup actually returns
    the user. The bound JWT is what enforces tenancy from this point on.
    """
    async with session.begin():
        await apply_admin_scope(session, on=True)
        yield session
```

It runs on the `app_admin` engine, which has its own policy — `admin_bypass`, which only opens up when `app.is_superadmin = 'on'` is explicitly set on the transaction:

```sql
CREATE POLICY admin_bypass ON customers
    AS PERMISSIVE FOR ALL TO app_admin
    USING (coalesce(current_setting('app.is_superadmin', true), 'off') = 'on')
    WITH CHECK (coalesce(current_setting('app.is_superadmin', true), 'off') = 'on');
```

Look at the `coalesce(..., 'off')`. **Same fail-closed instinct.** Setting absent → `'off'` → policy fails → zero rows. Even the god-mode role defaults to seeing nothing until it explicitly asks for god mode inside a transaction.

The critical discipline: `auth_bootstrap_session` is used by the auth handshake and *nowhere else*. The moment the OTP is verified and a JWT is issued, every subsequent request goes back through `get_tenant_session` and the normal RLS path. The escape hatch is three endpoints wide, and it's documented in the function's own docstring so nobody wanders into it by accident.

---

## The Tables That Don't Fit

Real schemas are never uniform, and pretending otherwise is how you get a policy that silently doesn't apply. Three special cases:

**`shops`** has no `shop_id` column — it *is* the shop. Its policy scopes on `id` instead, plus a soft-delete check:

```sql
USING (id = nullif(current_setting('app.current_shop_id', true), '')::uuid
       AND deleted_at IS NULL)
```

**`users`** can legitimately have `shop_id IS NULL` — that's a super-admin, who belongs to no shop. A blanket tenant policy would either hide them from themselves or, done carelessly, expose them to everyone. It gets a bespoke policy.

**`otp_requests`** and `refresh_tokens` are touched *before* a tenant scope exists — they're part of the bootstrap problem above. They get `GRANT`s but **no RLS**, and the application enforces scoping in the `WHERE` clause the old-fashioned way. That's a real, conscious weakening, so it's written down explicitly in the migration's docstring rather than left for someone to discover:

> `otp_requests` and `refresh_tokens` are auth tables touched pre-tenant — they get GRANTs but no RLS, and the application enforces scoping in WHERE clauses.

If you're going to have an exception, *name* it. An undocumented exception is just a bug with tenure.

---

## Proving It, Because "I Wrote a Policy" Is Not Evidence

RLS is exactly the kind of thing that is either working perfectly or not working at all, and looks identical either way from the application's perspective. A policy with a typo, a table you forgot to `FORCE`, a role that quietly has `BYPASSRLS` — all of them produce an app that works great and protects nothing.

So there are two dedicated test suites:

- **`test_rls_direct.py`** — connects to Postgres as `app_user`, sets `app.current_shop_id` to Shop A, and asserts that Shop B's rows are invisible. No application code in the loop at all. This tests the *database*, which is the thing making the promise.
- **`test_cross_tenant_isolation.py`** — drives the actual API with Shop A's JWT and hammers at Shop B's resources by ID. Every endpoint. Expects nothing back.

The second suite is the one that catches the mistakes that matter, because it tests the guarantee the way an attacker would attack it: not "does the filter work" but "can I get *anything* I shouldn't."

---

## What I'd Tell You

**1. `ENABLE` is not enough — you need `FORCE`.** `ENABLE ROW LEVEL SECURITY` exempts the table owner, and the table owner is usually whoever ran your migrations, which is usually whoever your app connects as. Without `FORCE`, your policies can be perfectly written and apply to literally nobody. This is the single most common way RLS silently does nothing.

**2. Your app role must have `NOBYPASSRLS`.** If you connect as a superuser, none of this exists. Check what your connection string is actually authenticating as before you write a single policy.

**3. Fail closed, deliberately.** `nullif(current_setting(...), '')` turns "scope not set" into zero rows instead of an error or — far worse — a missing filter. Design the failure mode first. The question isn't "does this work when everything goes right," it's "what does an attacker see when something goes wrong."

**4. Transaction-local, not session-local.** `set_config(..., true)` or `SET LOCAL`. A session-level tenant scope on a pooled connection is a cross-tenant leak with extra steps, and it will only show up under load, in production, intermittently. Which is to say: it will show up in the worst possible way.

**5. Solve the bootstrap problem on purpose.** Your login flow *must* read rows before it knows who the user is. You will need an escape hatch. Make it explicit, make it narrow, document it in the code, and make sure it fails closed too.

**6. Test the database, not just the app.** Write at least one test that connects directly as your application role and tries to read another tenant's rows. If that test would still pass with all your policies dropped, it isn't testing anything.

---

## The Point

The `WHERE shop_id = :shop_id` model asks me to be perfect forever. Every query, every join, every script, every 1am hotfix, every agent-written endpoint. One miss and a shop owner sees a competitor's debtors.

RLS asks the database to be perfect instead, once, in migration 1.

I know which of us I trust more.

---

**Tags:** #Postgres #RLS #MultiTenancy #Security #FastAPI #SaaS #Database
