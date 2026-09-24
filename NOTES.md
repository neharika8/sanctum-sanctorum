\# Notes



\*\*Live URL:\*\* https://sanctum-sanctorum-9f6f.onrender.com

\*\*API docs:\*\* https://sanctum-sanctorum-9f6f.onrender.com/docs



> Note: this is a free-tier Render instance. It spins down after inactivity, so the first

> request after a period of idle time can take up to \~50 seconds to respond. The attached

> Postgres database is also on Render's free plan and expires 30 days after creation.



\## What's done



All endpoints described in SPEC.md are implemented and passing the full test suite

(`uv run pytest` → 202 passed). This covers:



\- \*\*Books:\*\* create, get, list (search by title/author, filter by restricted/min\_price/max\_price,

&#x20; sort, pagination), partial update (PATCH)

\- \*\*Members:\*\* create (with duplicate-email rejection), get, stats (orders paid, spend, loan

&#x20; activity, late fees)

\- \*\*Orders:\*\* create (with restricted-book and stock checks, tier + bulk discounts, price

&#x20; snapshotting), pay, cancel (with stock restoration), list by member

\- \*\*Loans:\*\* borrow, return (with late-fee calculation), get, list by member with status

&#x20; filtering, tier-based loan limits, overdue blocking

\- \*\*Reports:\*\* top-selling books by paid order volume



\## What's skipped



Nothing from SPEC.md was intentionally skipped. The starting codebase already had `books.py`

and `schemas.py` fully implemented; the remaining gaps (loans, orders, member stats, reports,

and a few bugs) are what I completed.



\## Decisions / spec ambiguities



\- \*\*Bulk-discount rounding:\*\* SPEC.md specifies `discount\_cents = subtotal \* percent // 100`

&#x20; (integer floor division), so a $9.99 book at a 5% discount yields 49 cents off, not 49.95 —

&#x20; implemented literally as specified.

\- \*\*All-or-nothing stock check on orders:\*\* `create\_order` loads every book and validates stock

&#x20; for every item \*before\* mutating anything, so a multi-item order either fully succeeds or fails

&#x20; with no partial stock changes. This is straightforward at this project's scale but isn't safe

&#x20; under concurrent requests without row-level locking (`SELECT ... FOR UPDATE`) — worth

&#x20; revisiting if this were a real production system with concurrent writers.

\- \*\*Late fee rounding:\*\* "any partial day counts" is implemented as `ceil()` of elapsed time

&#x20; since `due\_at`, so 1 second late still counts as a full day's fee, matching SPEC.md's examples.



\## Git history



I was given the starting code as a zip file, not a git repository, so there is no prior commit

history to preserve or diff against. My repository's history therefore begins with my own first

fix rather than the unmodified starting point — each commit represents one area of the codebase

I completed (model, members, orders, loans, reports), followed by a final commit adding the

remaining files that were already correct in the zip and needed no changes.



\## AI usage



I used Claude (Anthropic) throughout this assignment: to understand what the assignment docs

were asking for, to identify what was missing vs. broken across the service files, to write the

implementations for the stubbed functions (loans, orders, member stats, reports), and to walk

through deployment (Render web service + Postgres) and Windows/PowerShell/git setup issues.



I reviewed and can explain every change. Two examples of places I double-checked or would push

back on:



\- Claude initially flagged `tier\_at\_least` using `>` instead of `>=` as a bug (a member exactly

&#x20; at the restricted-book minimum tier was being wrongly blocked). I verified this against

&#x20; SPEC.md's wording ("at or above") and confirmed it was a real bug, not a style preference.

\- Claude's `create\_order` implementation loads every book into a dict up front and does the

&#x20; stock check as a separate pass before mutating anything. I'd want to revisit this under

&#x20; concurrent load (see "all-or-nothing stock check" above) — it's correct for the single-request

&#x20; case the tests cover, but not safe if two people order the last copy of a book at the same time.

