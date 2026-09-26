\# Notes



\*\*Live URL:\*\* https://sanctum-sanctorum-9f6f.onrender.com  

\*\*API docs:\*\* https://sanctum-sanctorum-9f6f.onrender.com/docs



> \*\*Note:\*\* This is a free-tier Render instance. It spins down after inactivity, so the first request after a period of idle time can take up to \~50 seconds to respond. The attached Postgres database is also on Render's free plan and expires 30 days after creation.



\## What's Done



All endpoints described in `SPEC.md` are implemented and passing the full test suite (`uv run pytest` → \*\*202 passed\*\*).



This covers:



\- \*\*Books:\*\* create, get, list (search by title/author, filter by restricted/min\_price/max\_price, sort, pagination), partial update (PATCH)

\- \*\*Members:\*\* create (with duplicate-email rejection), get, stats (orders paid, spend, loan activity, late fees)

\- \*\*Orders:\*\* create (with restricted-book and stock checks, tier + bulk discounts, price snapshotting), pay, cancel (with stock restoration), list by member

\- \*\*Loans:\*\* borrow, return (with late-fee calculation), get, list by member with status filtering, tier-based loan limits, overdue blocking

\- \*\*Reports:\*\* top-selling books by paid order volume



\## What's Skipped



Nothing from `SPEC.md` was intentionally skipped.



The starting codebase already had `books.py` and `schemas.py` fully implemented. The remaining gaps — loans, orders, member stats, reports, and a few bugs — are what I completed.



\## Decisions / Spec Ambiguities



\- \*\*Bulk-discount rounding:\*\* `SPEC.md` specifies `discount\_cents = subtotal \* percent // 100` (integer floor division), so a $9.99 book at a 5% discount yields 49 cents off, not 49.95 cents. This was implemented literally as specified.



\- \*\*All-or-nothing stock check on orders:\*\* `create\_order` loads every book and validates stock for every item \*before\* mutating anything, so a multi-item order either fully succeeds or fails with no partial stock changes. This is straightforward at this project's scale but isn't safe under concurrent requests without row-level locking (`SELECT ... FOR UPDATE`). This would be worth revisiting in a real production system with concurrent writers.



\- \*\*Late fee rounding:\*\* "Any partial day counts" is implemented as `ceil()` of elapsed time since `due\_at`, so 1 second late still counts as a full day's fee, matching the examples in `SPEC.md`.



\## Git History



I was given the starting code as a ZIP file, not a Git repository, so there was no prior commit history to preserve or diff against.



My repository's history therefore begins with my first fix rather than the unmodified starting point. Each commit represents one area of the codebase I completed — model, members, orders, loans, and reports — followed by a final commit adding the remaining files that were already correct in the ZIP and needed no changes.



\## AI Usage



I used Claude (Anthropic) throughout this assignment to:



\- Understand what the assignment documentation was asking for

\- Identify what was missing or broken across the service files

\- Write implementations for the stubbed functions (loans, orders, member stats, reports)

\- Walk through deployment (Render web service + Postgres)

\- Resolve Windows/PowerShell/Git setup issues



I reviewed and can explain every change.



Two examples of places I double-checked or would push back on:



\- \*\*`tier\_at\_least` condition:\*\* Claude initially flagged `tier\_at\_least` using `>` instead of `>=` as a bug, because a member exactly at the restricted-book minimum tier was being wrongly blocked. I verified this against `SPEC.md`'s wording ("at or above") and confirmed it was a real bug, not a style preference.



\- \*\*`create\_order` stock handling:\*\* Claude's implementation loads every book into a dictionary up front and performs the stock check as a separate pass before mutating anything. I'd want to revisit this under concurrent load (see "All-or-nothing stock check" above). It is correct for the single-request case covered by the tests, but it is not safe if two people order the last copy of a book at the same time.

