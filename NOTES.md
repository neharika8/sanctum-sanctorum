# Notes

**Live URL:** https://sanctum-sanctorum-9f6f.onrender.com
**API docs:** https://sanctum-sanctorum-9f6f.onrender.com/docs

> **Note:** This is a free-tier Render instance. It spins down after inactivity, so the first request after a period of idle time can take up to ~50 seconds to respond. The attached Postgres database is also on Render's free plan and expires 30 days after creation.

## What's Done

All endpoints described in `SPEC.md` are implemented and passing the full test suite (`uv run pytest` → **202 passed**).

This covers:

- **Books:** create, get, list (search by title/author, filter by restricted/min_price/max_price, sort, pagination), partial update (PATCH)
- **Members:** create (with duplicate-email rejection), get, stats (orders paid, spend, loan activity, late fees)
- **Orders:** create (with restricted-book and stock checks, tier + bulk discounts, price snapshotting), pay, cancel (with stock restoration), list by member
- **Loans:** borrow, return (with late-fee calculation), get, list by member with status filtering, tier-based loan limits, overdue blocking
- **Reports:** top-selling books by paid order volume

## What's Skipped

Nothing from `SPEC.md` was intentionally skipped.

Every file in `app/` had gaps when I received it — either `TODO` comments or `NotImplementedError` stubs. This included:

- `app/schemas.py`: the ISBN-13 checksum was a stub that only checked digit count, not the actual check digit; the email validator didn't strip/lowercase before validating; `OrderCreate` had no validation against empty or duplicate items.
- `app/services/books.py` / `app/routers/books.py`: no duplicate-ISBN check, no `PATCH` endpoint at all, and `list_books` was missing `min_price`/`max_price` filtering, `sort`, and had a bug computing `total` from the paginated slice instead of the full filtered set.
- `app/services/members.py`, `app/services/orders.py`, `app/services/loans.py`, `app/services/reports.py`: fully unimplemented (member stats, order creation/pricing, loan borrowing/returns/late fees, top-books report).

I completed all of the above.

## Decisions / Spec Ambiguities

- **Bulk-discount rounding:** `SPEC.md` specifies `discount_cents = subtotal * percent // 100` (integer floor division), so a $9.99 book at a 5% discount yields 49 cents off, not 49.95 cents. This was implemented literally as specified.

- **All-or-nothing stock check on orders:** `create_order` loads every book and validates stock for every item *before* mutating anything, so a multi-item order either fully succeeds or fails with no partial stock changes. This is straightforward at this project's scale but isn't safe under concurrent requests without row-level locking (`SELECT ... FOR UPDATE`). This would be worth revisiting in a real production system with concurrent writers.

- **Late fee rounding:** "Any partial day counts" is implemented as `ceil()` of elapsed time since `due_at`, so 1 second late still counts as a full day's fee, matching the examples in `SPEC.md`.

## Git History

I was given the starting code as a ZIP file rather than a Git repository, so there was no prior commit history to preserve. My history begins with my own first fix.

Most commits map to one feature area: the Loan model, member fixes (tier check, duplicate email, stats), orders (discounts, stock, cancel fix), loans (borrowing/returns/late fees), and reports. One commit, `Add remaining project files`, is broader than the rest and bundles several files together — including real fixes to `app/schemas.py` (ISBN-13 checksum, email normalization, order-item validation) and `app/routers/books.py`/`app/services/books.py` (duplicate-ISBN check, the `PATCH` endpoint, and search filters/sort/pagination). In hindsight I'd have split that into its own dedicated "Books" commit to keep one clean diff per feature, matching the pattern of the other commits.

## AI Usage

I used Claude (Anthropic) throughout this assignment to:

- Understand what the assignment documentation was asking for
- Identify what was missing or broken across the service files
- Write implementations for the stubbed functions (books, schemas, members, orders, loans, reports)
- Walk through deployment (Render web service + Postgres)
- Resolve Windows/PowerShell/Git setup issues

I reviewed and can explain every change.

Two examples of places I double-checked or would push back on:

- **`tier_at_least` condition:** Claude initially flagged `tier_at_least` using `>` instead of `>=` as a bug, because a member exactly at the restricted-book minimum tier was being wrongly blocked. I verified this against `SPEC.md`'s wording ("at or above") and confirmed it was a real bug, not a style preference.

- **`create_order` stock handling:** Claude's implementation loads every book into a dictionary up front and performs the stock check as a separate pass before mutating anything. I'd want to revisit this under concurrent load (see "All-or-nothing stock check" above). It is correct for the single-request case covered by the tests, but it is not safe if two people order the last copy of a book at the same time.
