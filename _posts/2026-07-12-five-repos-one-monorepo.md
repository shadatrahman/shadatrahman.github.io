---
title: "Five Repos, One Monorepo — And Why My AI Agents Made Me Do It"
date: 2026-07-12 00:00:00 +0600
categories: [AI, Development, Architecture]
tags: [monorepo, git, ai, claude-code, architecture, developer-experience, fastapi, nextjs, flutter]
description: I split Amar Khata into five clean repos like a responsible adult. Then I read my own git history and realized I'd been hand-carrying context between them for a month. Here's how I merged them into one monorepo without losing a single commit—and why the agent's context window is the new module boundary.
---

I did everything right, and it was wrong.

I'm building **Amar Khata** (আমার খাতা) — a Bangla-first SaaS for Bangladeshi shop owners to track daily sales and customer dues (বাকি / *baki*). One FastAPI backend, a Next.js web app for shop owners, a Flutter mobile app, a Next.js admin panel for me, and a shared brand package. Five clean, well-separated concerns.

So I gave each of them its own repository. Of course I did. That's what you do. Separate deploy pipelines, separate CI, separate issue trackers, separate release cadences. Every architecture blog post I've ever read nodded along approvingly.

A month later I sat down to add double-entry bookkeeping — a proper general ledger with a trial balance — and I finally admitted what my git history had been screaming at me the whole time.

The five-repo split wasn't organizing my code. It was organizing my *pain*.

---

## The Tell: Reading Your Own Git Log Like a Crime Scene

Here's the thing about working with AI agents every day: they make your architectural mistakes *legible*. A bad boundary that a human team would quietly route around with a Slack message becomes, with an agent, a very loud and very expensive failure. The agent can't Slack anybody. It can only see what's in the repo you pointed it at.

So when I went back and read my own commit history across the five repos, the evidence was damning. Let me show you three exhibits.

### Exhibit A: Commit messages carrying passports

```
feat(staff): multi-staff invites + permissions + role gating (Amar-Khata/mobile_app#21+#22) (#48)
feat(share): public read-only customer dues page (Amar-Khata/web_app#24) (#42)
feat(share): public signed read-only customer dues link (Amar-Khata/backend#5) (#15)
feat(suppliers,payables,payments-out): supplier-side ledger (Amar-Khata/backend#6) (#14)
```

Look at those. Every single one has **two** issue numbers: a local one, and a fully-qualified foreign one from a *different repository*. `Amar-Khata/mobile_app#21`. `Amar-Khata/backend#6`.

That's not a commit message. That's a customs declaration. Every feature had to be manually escorted across a repo border, and I was the one carrying its papers. One feature — "let a shop owner share a read-only dues link with a customer" — existed as an issue in the backend repo, *another* issue in the web repo, and a third in mobile. Three issues. Three PRs. Three reviews. One feature.

### Exhibit B: A commit whose entire job was copy-paste

```
chore(l10n_bn): mirror suppliers/payables/paymentsOut keys from web
```

Read that one again. The complete and total purpose of that commit was to copy Bangla translation strings *from one repository into another repository* by hand.

Amar Khata is Bangla-first — not Bangla-optional. Every string, every validation message, every SMS body is Bangla. Web keeps its strings in `web_app/src/i18n/bn.ts`. Mobile keeps the same key set in `mobile_app/lib/core/l10n_bn.dart`. They must stay in lockstep, or a shop owner sees a raw key like `suppliers.payable.title` where a Bangla label should be.

In a polyrepo, keeping those two files in sync is a *ritual*. And I performed it, by hand, in a commit, like an animal.

### Exhibit C: I built a tool to solve a problem I had personally created

```
feat(i18n): Bangla strings drift detector — closes #28
```

This is the one that actually stung.

I wrote an entire drift detector — real code, tests, CI wiring — whose only reason to exist was that my translation keys lived in two repos that couldn't see each other. I was so deep in the polyrepo mindset that when the boundary caused a problem, my instinct was to **build tooling to police the boundary** rather than to ask whether the boundary should exist.

That's the polyrepo tax, and I'd been happily paying it. If the two files had lived in one repo, the "drift detector" would have been a diff. Or nothing at all, because the agent editing one would have simply seen the other sitting right there.

### The bonus exhibit: my agents were being cloned

There's a fourth piece of evidence that only shows up in the AI era. Every repo had its own agent configuration:

```
backend/CLAUDE.md          backend/.cursorrules       backend/.mcp.json
backend/AGENTS.md          backend/.windsurfrules     backend/.opencode.json
backend/.claude/skills/fastapi-developer/SKILL.md          ← 894 lines
backend/.claude/skills/github-project-manager/SKILL.md     ← 1,077 lines
```

...and then the same seven-file set again in `web_app/`, and again in `admin_panel/`, and again in `mobile_app/`. An 894-line skill file, duplicated. A 1,077-line skill file, duplicated. Every time I improved how my agent understood the project, I had to improve it **five times** — or, more realistically, improve it once and let the other four rot.

Five repos meant five agents, each with a lobotomized view of a system that is fundamentally *one system*.

---

## The Real Reason: The Context Window Is the New Module Boundary

Here's the idea I want you to take away from this post, even if you skip the rest.

We choose repo boundaries based on **deploy** boundaries. Backend deploys separately from mobile, so: separate repos. That logic is decades old and it's still not *wrong*, exactly. But it optimizes for a constraint that has quietly stopped being the binding one.

The binding constraint now is: **what can a single agent hold in its head at once?**

Because a feature is not a deploy unit. A feature is a *vertical slice*. Take the one that finally broke me — the trial balance:

- A Postgres migration and a new `accounting` module in the **backend**
- Two new endpoints on the **API contract**
- A report page with an as-of-date picker in the **web app**
- A tenant-inspection view in the **admin panel**
- Bangla strings in **two different i18n files**
- Design tokens from the **brand package**

That's five repos for *one feature*. In a polyrepo, an agent can see exactly one-fifth of the problem at any moment. So I became the integration layer. I'd have Claude build the backend endpoint, then I'd copy the response shape into my head, walk it over to the web repo, open a fresh session, and re-explain the context I'd just destroyed by closing the last one.

I was a very expensive, very slow message bus between five context windows.

The monorepo isn't about the build system. I don't have a fancy Bazel setup. There's no shared node_modules wizardry. **The monorepo is about letting one agent see one feature.** That's it. That's the whole justification, and it turned out to be more than enough.

---

## The Merge: Five Repos, Zero Lost Commits

Right. So how do you actually do this without torching a month of history?

The tempting shortcut is to just copy the files in:

```bash
# ❌ Don't do this
cp -r ../backend ./backend
git add . && git commit -m "add backend"
```

This "works" and it is a disaster. You get one giant commit that claims you wrote 40,000 lines on a Tuesday. `git blame` points at that commit for every line. `git log` on any file shows exactly one entry. Every "why on earth is this code like this?" question you will ever ask becomes permanently unanswerable. You have deleted your project's memory to save an afternoon.

The right way takes about twenty minutes and preserves everything.

### Step 1: Rewrite each repo's history into a subdirectory

The trick is that before you merge anything, you rewrite each source repo so that **every commit in its entire history** looks like it always lived in a subdirectory. Not just the current files — every commit, all the way back to the initial one.

[`git-filter-repo`](https://github.com/newren/git-filter-repo) does this in one flag:

```bash
# Work on a throwaway clone — filter-repo rewrites history irreversibly
git clone git@github.com:Amar-Khata/backend.git backend-import
cd backend-import

# Rewrite EVERY commit so its tree lives under backend/
git filter-repo --to-subdirectory-filter backend
```

Before this, the initial commit's tree looked like `alembic/  src/  pyproject.toml`. After it, that same initial commit's tree is `backend/alembic/  backend/src/  backend/pyproject.toml`. The commit SHAs change (that's unavoidable — you're rewriting trees), but the authors, dates, messages, and parent relationships all survive intact.

You can verify it took by checking the tree of the *oldest* commit in the rewritten history:

```bash
git ls-tree --name-only $(git rev-list HEAD | tail -1)
# backend        ← the very first commit now lives under backend/ too
```

That single line is the whole ballgame. If you see `backend` there, every commit in that repo's life has been relocated and your merge is going to be conflict-free.

Repeat for each repo — `--to-subdirectory-filter web_app`, `mobile_app`, `admin_panel`, `brand`.

### Step 2: Merge unrelated histories

Now the merge itself. Each rewritten repo has a history that shares no common ancestor with your monorepo, so Git will refuse by default — you have to explicitly tell it you know what you're doing:

```bash
cd ../amar-khata

git remote add backend-import ../backend-import
git fetch backend-import

git merge backend-import/main \
  --allow-unrelated-histories \
  -m "merge: import backend with full history"

git remote remove backend-import
```

Because every incoming commit only ever touches paths under `backend/`, and nothing already in the monorepo touches `backend/`, **there are no conflicts.** The trees are disjoint. The merge is trivial. It just works.

Do that five times and your history looks like this:

```
*   3ec9414 merge: import brand with full history
|\
| * 03fb169 ds
| * fae40e3 logo
| * 5c4e9ec init
*   bbe5295 merge: import web_app with full history
|\
| * 6efea6b wip: format helpers snapshot
| * b78489d feat(suppliers): supplier-side ledger — list, payable + payment-out drawers
| * 1bec990 feat(share): public read-only customer dues page
| * ...
*   fd26303 merge: import mobile_app with full history
|\
| * ef17b45 wip: format helpers + ios pod snapshot
| * a9e2cac feat(staff): multi-staff invites + permissions + role gating
| * ...
*   f694ac1 merge: import backend with full history
|\
| * 168f51b wip: suppliers/payables, share links, idempotency, sms snapshot
| * ...
*   cfa5793 merge: import admin_panel with full history
```

Five braids, one rope. Every commit that ever existed in any of those repos is still there, still attributed, still dated, still explaining itself. Here's `--follow` on a file that came in through the merge:

```console
$ git log --follow --format='%h %ad %s' --date=short -- backend/src/modules/sales/router.py
c547caa 2026-06-04 feat(backend): double-entry GL + trial balance
c29c12d 2026-05-07 feat(sales,payments): owner-only PATCH + DELETE with total_due reconciliation
7e44c93 2026-05-05 feat: sales phase — customers/sales/payments CRUD + idempotency + reconciler
```

The merge happened on 2026-06-04. Those bottom two commits are from **May** — they're the old backend repo's history, and Git walks straight back into them without blinking. `git blame` still names the commit that introduced any given line. **Nothing was lost.**

Final tally: 86 commits in the monorepo, 65 of them inherited — 27 from backend, 19 from mobile, 12 from web, 4 from admin, 3 from brand.

### The honest part

Look closely at those pre-merge tips and you'll notice something less than heroic:

```
168f51b wip: suppliers/payables, share links, idempotency, sms snapshot
ef17b45 wip: format helpers + ios pod snapshot
6efea6b wip: format helpers snapshot
```

Three `wip: ... snapshot` commits, made the morning of the merge.

I had uncommitted work-in-progress sitting in three of the five repos when I decided to do this. `filter-repo` works on committed history, not your dirty working tree. So rather than lose it or spend a day untangling it into clean commits, I snapshotted it. Ugly commits, honestly labelled, preserved forever in the history.

I'd rather have an honest `wip:` commit in my log than a lie. If you attempt this migration, expect to make a few of these. Label them clearly and move on — the alternative is stalling the migration for a week while you groom branches, and the migration is worth more than the grooming.

---

## The Root Contract: Where the Real Value Landed

Merging the code was the easy part, and honestly it isn't where the value came from. The value came from what became *possible* once everything was in one tree: **a single document that tells any agent how this system actually works.**

Two files at the root, added the same day:

**`CLAUDE.md`** — the cross-cutting contract. Not a README. A contract. The thing I was previously re-explaining out loud to every fresh agent session, written down once. The most important parts are the non-negotiables, stated flatly:

> **Backend owns truth. Clients are online-only for v1** — no offline-first sync. The web/admin apps assume connectivity; mobile shows an offline banner and disables writes when the network drops. Do not add a local write queue.

> **Every business row is tenant-scoped.** Tenant = `Shop`. Postgres RLS is the primary defense from migration 1; the repo `shop_id` arg is defense-in-depth, never the sole gate. If a query can run without a `shop_id` filter, it's a bug.

> **Bangla-first, not Bangla-optional.** All UI strings, validation, SMS bodies are Bangla. Numerals render in Bangla digits (০১২৩…) by default. No inline copy — strings come from the per-app i18n file.

Every one of those blockquotes is a rule an agent would otherwise cheerfully violate. Ask a competent model to "add offline support so the app works on bad networks" and it will *happily* build you a local write queue, because that's a good idea in general and it has no way to know it's a catastrophic idea *here* (a shop owner's phone silently replaying a sale twice is a wrong ledger, and a wrong ledger is a lost customer).

The rest of the contract is the stuff that's cheap to state and expensive to get wrong:

- **Money**: `Decimal` / `Numeric(12,2)` only, never float. `total_due` is denormalized — update it atomically in the same transaction with `UPDATE … SET total_due = total_due + :delta`, never read-modify-write.
- **Time**: stored UTC, presented `Asia/Dhaka`. Day boundaries computed in the shop's timezone.
- **Idempotency**: every mutating write takes an `Idempotency-Key`. Same key + same body → replay. Same key + different body → 422. **No exceptions.**
- **The API is the one contract.** It's OpenAPI-first: FastAPI publishes the spec, and all three clients *generate* their typed bindings from it. Never hand-write a request/response type. `operation_id`s are stable and snake_case; renaming one is a breaking change. Drift is CI-gated.

**`TRACKER.md`** — the living, one-task-at-a-time plan. A "Now" pointer at the top, status tables per area, and a cross-cutting follow-ups list. Its whole job is to answer "what's the state, what's next?" in one place instead of across five issue trackers that never agreed with each other.

Between them, these two files do something the five READMEs never could: they let an agent — or a new contributor, or *me* on a Monday — load the entire system's rules in one read.

---

## The Payoff: A Feature That Crossed Three Clients in One Branch

So: back to double-entry bookkeeping. The feature that started all this.

A trial balance is not a small ask. It means a real general ledger: chart of accounts, journal entries, opening balances, and the iron guarantee that debits equal credits. It has to project the operational tables (sales, payments, payables) as subledgers. The shop owner needs a report page. I need to inspect any tenant's books from the admin panel.

In the polyrepo, this was a multi-day cross-repo coordination problem. Here's what it actually looked like after the merge — one branch, `feat/trial-balance`:

```
c547caa  feat(backend): double-entry GL + trial balance (accounts, journal, opening balances)
4babad9  feat(web): trial balance report page + journal entry + opening balances
bfabc5c  feat(admin): view any tenant's trial balance at any date on the tenant page
e3a32de  docs(tracker): record Trial Balance (double-entry GL) shipped across all clients
```

Backend, then web, then admin, then update the plan. **One branch. One afternoon. One continuous agent session that never lost the thread.**

The backend commit touched 20 files under `backend/` — a new `accounting` module, a migration, a `DEFERRABLE` Postgres constraint trigger that makes an unbalanced journal entry literally impossible to commit, and ten dedicated engine tests. Then, without me re-explaining a single thing, the same session built the web report page against the endpoint *it had just written* — no spec hand-off, no copying response shapes between context windows, no "here's what the API returns, trust me." Then the admin view.

That's the difference. Not "the build is faster." The *feature* is faster, because the feature stopped being five conversations.

---

## What the Monorepo Did Not Fix

I want to be straight with you here, because monorepo posts have a bad habit of ending at the victory lap.

**It didn't delete the API contract — it made it more important.** Putting the backend and the clients in one folder does not mean the client can reach into the backend's internals. The OpenAPI spec is still the only legal way across that line, and it's still CI-gated. A monorepo without a contract is just a big ball of mud with better ergonomics. If anything, the temptation to cheat is *higher* now that the backend source is right there, one directory over.

**Codegen drift is still manual.** Right now `pnpm gen:api` and the Dart generator are run by hand. The monorepo makes the drift *visible* — you can finally diff a spec change against its clients in one place — but it doesn't auto-regenerate anything. That's still on my follow-up list, and the honest version is: a monorepo turned an invisible problem into a visible one. That's an upgrade, not a solution.

**The agent config is still duplicated.** Here's a genuinely funny one. Right now, today, my repo contains:

```
.claude/skills/fastapi-developer/SKILL.md          ← 894 lines
backend/.claude/skills/fastapi-developer/SKILL.md  ← the same 894 lines
```

The merge preserved the duplication I was trying to escape. Of *course* it did — it preserved everything, that was the entire point. Deduplicating the agent config is a follow-up I haven't done yet, and I'm leaving it in this post rather than quietly cleaning it up before publishing, because it's the honest shape of a migration: the merge is a beginning, not an ending.

**Stale paths outlive the move.** My subproject docs still refer to `backend_amar_khata/` and `frontend_amar_khata/web/` — paths that no longer exist anywhere. Grep for your old repo names after a merge. You'll find them for months.

**CI needs path filters, or it gets dumber and slower.** Every push now potentially triggers everything. You need path-based filters so a Flutter change doesn't run the pytest suite. This is a real cost, and it's the one legitimate thing the polyrepo gave me for free.

---

## What I'd Tell You

**1. Repo boundaries should follow context boundaries, not deploy boundaries.** Your services can still deploy independently from a single repo — that's a CI config, not a repo layout. But your *agent* cannot reason across a boundary it cannot see. Ask yourself which of your boundaries exist for good reasons and which exist because you read a blog post in 2019.

**2. Your git log is an honest architecture review.** If your commit messages carry foreign issue numbers, you have a boundary problem. If a commit's entire job is copying a file from another repo, you have a boundary problem. If you've built tooling whose only purpose is detecting drift between two repos, you have *definitely* got a boundary problem, and you have expensively institutionalized it. Go read your history. It's been trying to tell you.

**3. Never lose history to save an afternoon.** `git filter-repo --to-subdirectory-filter` plus `git merge --allow-unrelated-histories` is a twenty-minute procedure that preserves every commit, every author, every date, and every `git blame` you will ever run. The `cp -r` version costs you your project's memory forever. There is no version of that trade that's worth it.

**4. A monorepo without a root contract is just a bigger mess.** The merge bought me the *possibility* of one agent seeing one feature. `CLAUDE.md` and `TRACKER.md` are what turned that possibility into something that actually works. Write down your non-negotiables — especially the ones where the globally-good idea is the locally-catastrophic one. "Do not add a local write queue" is not obvious to anyone, human or model, who hasn't been burned by it. So write it down.

**5. Merge honestly.** `wip: snapshot` commits, stale paths, duplicated config that survived the move — all of it stays in my history. A migration that pretends to be clean is a migration that's hiding something from the next person to read the log. That person is usually you.

---

## Should You Do This?

Not always. The polyrepo is genuinely right when your services have **truly independent lifecycles and separate teams who don't coordinate**. If the backend team ships on their own cadence and the mobile team consumes a versioned, stable, public API — keep your repos. That boundary is real, it's staffed, and the cost of crossing it is a feature, not a bug.

But if you're a small team, or a solo builder, or anyone shipping **vertical features across a stack you own end to end** — and *especially* if you're working with AI agents daily — then that five-repo split isn't organizing your code.

It's just making you the message bus.

I spent a month hand-carrying context between five context windows and calling it good architecture. The trial balance took one afternoon after I stopped.

---

**Tags:** #Monorepo #Git #AI #ClaudeCode #Architecture #FastAPI #NextJS #Flutter
