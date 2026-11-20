# Money Journal — Design

Date: 2026-11-20
Status: approved by owner 2026-11-20

## Intent and why

A personal money journal, used by one person (the owner). Each month the owner
wants to know two things at a glance:

1. What their fixed costs add up to this month.
2. What they have left to spend this month.

They come back during the month to log spending, so "left" goes down as they
go. It must work from any device (phone, Mac, anything with a browser).

The owner is also building this to **learn Next.js**. The code should be
idiomatic, readable Next.js that teaches the framework, not a pile of
abstractions.

## Goals

- See "left this month" the moment the app opens, on phone or desktop.
- Log a spending entry in a few taps on a phone.
- Keep a list of recurring fixed costs that carries into each new month.
- Look back at any past month.
- Deployed on Vercel free tier; reachable from anywhere; only the owner can get in.

## Non-goals (v1)

- More than one user, sharing, or households.
- Bank / card sync or importing statements.
- Spending categories, per-category budgets, charts, reports.
- Multiple currencies.
- Offline mode / installable PWA.
- Native mobile app.

## Anti-goals (a failure even if it "works")

- Logging an entry on the phone is slow or fiddly, so the owner stops using it.
- Someone other than the owner can see the data.
- Data is lost, or past months silently change when a fixed cost is edited.
- Code is over-engineered so it stops being a good way to learn Next.js.

## Constraints

- Hosting: Vercel (free). No persistent disk, so no SQLite; data lives in a hosted DB.
- Single language: TypeScript.
- Free tiers only.

## Approaches considered

| | Approach | Trade-off |
|---|---|---|
| **A (picked)** | Next.js only (App Router, server actions), Prisma + Neon Postgres, Auth.js | One repo, one deploy, all free, fast on phone. No NestJS practice. |
| B | Next.js on Vercel + NestJS on a second host (e.g. Render free) | NestJS practice, but two deploys, CORS, cross-server auth, and free backend sleeps (30–60s cold start). |
| C (dropped) | NestJS as a Vercel serverless function | Possible but fights how both tools work. |

The owner picked **A** because they want to learn Next.js.

## Decided by the owner

- Web app usable on every device.
- Deploy on Vercel free tier.
- Next.js only, no separate backend.
- It is a journal: logged during the month, not a one-off calculator.
- **The owner writes the code; Claude teaches.** The plan is a series of small
  lessons. Each lesson teaches one Next.js idea (routing, server components,
  server actions, Prisma, Auth.js, deploy), explains why, and ends with a
  working app the owner can run. Claude explains and reviews; the owner types.
- Income is **one number per month** (`Month.income`), not a list of entries.
- Login is **Sign in with Google**, limited to the owner's email.

## Builder's design (proposed, owner may correct)

### Stack

- Next.js (current stable, App Router), TypeScript, React Server Components.
- Mutations via **server actions**; no separate REST API unless needed.
- Tailwind CSS, mobile-first layout.
- Prisma ORM, Neon Postgres (Vercel Marketplace integration, free tier).
- Auth.js (NextAuth v5), Google provider. Only the email in env var
  `ALLOWED_EMAIL` may sign in; everyone else is rejected.

### Core rule

```
left this month = income − sum(fixed costs this month) − sum(expenses this month)
```

"This month" is the calendar month in the owner's timezone (set by env var
`APP_TIMEZONE`). Left may go negative; show it in red, never block.

### Data model

Money is stored as integers in the currency's minor unit (e.g. cents/satang),
never floats. One currency, set by env var `APP_CURRENCY`, displayed with
`Intl.NumberFormat`.

- **FixedCostTemplate** — the owner's standing list: `name`, `amount`, `active`, `sortOrder`.
- **Month** — `year`, `month`, `income`. Unique on (year, month).
- **MonthFixedCost** — snapshot of a fixed cost for one month: `monthId`,
  `name`, `amount`. Created by copying active templates when a month is
  first opened. Editing a template changes future months only; editing a
  month's snapshot changes only that month. This keeps past months stable.
- **Expense** — `monthId`, `date`, `amount`, `note` (optional, short text).

No `userId` columns: single user, guarded by the auth allowlist.

### Month creation

When the owner opens a month that has no `Month` row yet, create it:
income defaults to the previous month's income (or 0), fixed costs copied
from active templates. Done lazily on first visit, not by a cron.

*(Builder's call)* Only the current month or past months can be created.
The › arrow stops at the current month, so browsing never creates a future
month early and locks its fixed costs. When the owner adds or edits a fixed-cost
template, the form has a checkbox "also apply to this month" (checked by
default), so a new subscription added mid-month shows in this month's Left.

### Export

*(Builder's call, for the "data is lost" anti-goal)* A "Download my data"
button on the History screen returns all months, fixed costs and expenses as
one JSON file. Neon's free-tier restore window is short, so this is the
owner's own backup.

### Screens

1. **This month (home)** — big "Left" number; under it income, fixed total,
   spent so far. Quick "add expense" form (amount, optional note, date
   defaults to today). List of this month's expenses, newest first, each
   editable/deletable.
2. **Month setup** — edit this month's income and its fixed-cost snapshot.
3. **Fixed costs** — manage the standing template list.
4. **History** — list of past months with income / fixed / spent / left;
   tap one to open it (same layout as home, for that month).
5. **Sign in** — Google sign-in button.

Rough phone layout of home:

```
┌─────────────────────────┐
│  November 2026     ‹ ›  │
│                         │
│      Left  12,340       │
│  Income 30,000          │
│  Fixed  −15,000         │
│  Spent   −2,660         │
│                         │
│ [ amount ][ note ] [Add]│
│                         │
│ 20 Nov  Lunch     −120  │
│ 19 Nov  Grab       −85  │
│ ...                     │
└─────────────────────────┘
```

### Validation and errors

- Validate every server action input on the server (amount positive integer
  within a sane max, note length cap, date within the month). Use `zod`.
- Every server action checks the session first.

### Testing

- Unit test the "left" calculation and the month-creation copy logic.
- One end-to-end smoke check after deploy: sign in, add an expense, see Left drop.

## Left to the builder

- Exact folder structure, component split, styling details.
- Whether to use Vitest or Jest for unit tests.
- Neon / Vercel setup steps (documented in README).
- Delete confirmations and empty states.
