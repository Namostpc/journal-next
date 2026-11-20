# Money Journal Implementation Plan (lesson format)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **For this project:** the owner writes the code. Claude teaches each lesson, answers questions, and reviews the owner's code against the lesson's tests and checks. Claude does not write the product code unless the owner asks.

**Goal:** A personal money journal on Vercel that shows "left this month" (income − fixed costs − spending) and lets the owner log spending from any device.

**Architecture:** One Next.js App Router app. Pages are server components that read Postgres through Prisma; every change goes through a server action. Auth.js (Google) lets only `ALLOWED_EMAIL` in. Pure money/month logic lives in `src/lib/` and is unit tested with Vitest.

**Tech Stack:** Next.js (latest stable, App Router, TypeScript), Tailwind CSS, Prisma + Neon Postgres, Auth.js v5 (Google provider), zod, Vitest.

**Spec:** `docs/superpowers/specs/2026-11-20-money-journal-design.md`

## Global Constraints

- TypeScript everywhere. Free tiers only (Vercel, Neon, Google OAuth).
- Money is stored as integers in minor units (cents/satang). Never floats.
- One currency from env `APP_CURRENCY` (e.g. `THB`), shown with `Intl.NumberFormat`.
- "This month" = calendar month in env `APP_TIMEZONE` (e.g. `Asia/Bangkok`). Vercel runs in UTC, so never use the server's local time.
- `left = income − sum(fixed costs this month) − sum(expenses this month)`. Left may be negative: show in red, never block.
- Only the email in env `ALLOWED_EMAIL` may sign in. Every page, server action and route handler checks the session.
- Validate every server action input on the server with zod.
- Editing a fixed-cost template never changes past months.
- No future months: the › arrow stops at the current month.
- Non-goals: multi-user, bank sync, categories, charts, multi-currency, offline, native app.
- Keep code plain and idiomatic: it is a learning project. No extra abstraction layers.
- GitHub repo goes under the **Namostpc** account, never namo443.

## Review Focus

1. **Odd amount input** (`"1,200"`, `"120.5"`, `"120.555"`, `"-5"`, `"0"`, `"abc"`, empty): accept the first two, reject the rest with a clear message. Pinned in Lesson 1 (`parseAmount` tests).
2. **Month boundary in the owner's timezone**: at 00:30 on 1 Dec in Bangkok, the server's UTC clock still says 30 Nov; the app must show December. Pinned in Lesson 1 (`currentYearMonth` tests).
3. **Expense date outside the month** (editing an expense to a date in another month): reject. Pinned in Lesson 5 (`expenseSchema` tests).
4. **A Google account that isn't the owner's**, and a server action called with no session: both rejected. Pinned in Lesson 8 (`isAllowedEmail` tests + manual check).
5. **Double-tap Add / two tabs opening a new month at once**: Add button disabled while pending; month creation survives the unique-constraint race. Pinned in Lesson 4 (race handling step) and Lesson 5 (pending button).

---

## File map

```
src/lib/money.ts          parseAmount, formatMoney
src/lib/month.ts          calcLeft, currentYearMonth, prevYearMonth, nextYearMonth, isAfter, buildNewMonth
src/lib/validation.ts     expenseSchema, incomeSchema, fixedCostSchema
src/lib/db.ts             prisma client singleton
src/lib/months.ts         getOrCreateMonth (DB)
src/lib/auth-rules.ts     isAllowedEmail
src/auth.ts               Auth.js config, requireOwner
prisma/schema.prisma      data model
src/app/page.tsx                          redirect to current month
src/app/signin/page.tsx                   Google sign-in button
src/app/api/auth/[...nextauth]/route.ts   Auth.js handlers
src/app/(app)/layout.tsx                  nav + requireOwner
src/app/(app)/m/[year]/[month]/page.tsx       month view (home)
src/app/(app)/m/[year]/[month]/setup/page.tsx month setup
src/app/(app)/fixed-costs/page.tsx        template list
src/app/(app)/history/page.tsx            past months
src/app/api/export/route.ts               JSON download
src/app/actions/expenses.ts               addExpense, updateExpense, deleteExpense
src/app/actions/month.ts                  updateIncome, saveMonthFixedCost, deleteMonthFixedCost
src/app/actions/templates.ts              saveTemplate, deactivateTemplate
src/components/*.tsx                      LeftSummary, ExpenseForm, ExpenseList, MonthNav, SubmitButton
tests/*.test.ts                           Vitest unit tests
```

---

### Lesson 0: Create the app

**You learn:** what `create-next-app` makes, the `app/` folder, the dev server, and how to run tests.

**Files:** whole new project in this folder; `vitest.config.ts`; `tests/smoke.test.ts`

- [ ] **Step 1:** Run `npx create-next-app@latest . --ts --tailwind --eslint --app --src-dir --import-alias "@/*"` (the `docs/` folder already here is fine).
- [ ] **Step 2:** Run `npm run dev`, open http://localhost:3000. Expected: Next.js starter page.
- [ ] **Step 3:** Replace `src/app/page.tsx` with a page showing the heading "Money Journal". Check in the browser.
- [ ] **Step 4:** `npm install -D vitest`, add `"test": "vitest run"` to `package.json`, add `vitest.config.ts` with the `@` alias pointing to `src`.
- [ ] **Step 5:** Write `tests/smoke.test.ts` with `expect(1 + 1).toBe(2)`. Run `npm test`. Expected: 1 passed.
- [ ] **Step 6:** Commit: `git add -A && git commit -m "chore: create Next.js app"`

---

### Lesson 1: Money and month logic (plain TypeScript, test first)

**You learn:** TypeScript types, test-driven development, why money is stored as integers, and time zones with `Intl`.

**Files:** Create `src/lib/money.ts`, `src/lib/month.ts`; Test `tests/money.test.ts`, `tests/month.test.ts`

**Interfaces (produces):**
```ts
// money.ts
export function parseAmount(input: string): number | null   // major-unit text -> minor int, null if invalid
export function formatMoney(minor: number, currency: string): string
export const MAX_AMOUNT = 1_000_000_000                      // minor units (10,000,000.00)

// month.ts
export type YearMonth = { year: number; month: number }     // month 1-12
export function calcLeft(income: number, fixedCosts: number[], expenses: number[]): number
export function currentYearMonth(timeZone: string, now?: Date): YearMonth
export function prevYearMonth(ym: YearMonth): YearMonth
export function nextYearMonth(ym: YearMonth): YearMonth
export function isAfter(a: YearMonth, b: YearMonth): boolean
export type Template = { id: number; name: string; amount: number; active: boolean }
export function buildNewMonth(prevIncome: number | null, templates: Template[]):
  { income: number; fixedCosts: { templateId: number; name: string; amount: number }[] }
```

- [ ] **Step 1: Write the failing tests** in `tests/money.test.ts`:
```ts
expect(parseAmount("120")).toBe(12000)
expect(parseAmount("120.5")).toBe(12050)
expect(parseAmount("1,200")).toBe(120000)
expect(parseAmount(" 99.99 ")).toBe(9999)
for (const bad of ["", "abc", "0", "-5", "120.555", "1e3", "10000000.01"]) expect(parseAmount(bad)).toBeNull()
expect(formatMoney(1234500, "THB")).toContain("12,345.00")
```
and `tests/month.test.ts`:
```ts
expect(calcLeft(3000000, [1500000], [12000, 8500])).toBe(1479500)
expect(calcLeft(100, [200], [])).toBe(-100)                       // negative allowed
expect(currentYearMonth("Asia/Bangkok", new Date("2026-11-30T17:30:00Z"))).toEqual({ year: 2026, month: 12 })
expect(currentYearMonth("Asia/Bangkok", new Date("2026-11-30T16:59:00Z"))).toEqual({ year: 2026, month: 11 })
expect(prevYearMonth({ year: 2026, month: 1 })).toEqual({ year: 2025, month: 12 })
expect(nextYearMonth({ year: 2026, month: 12 })).toEqual({ year: 2027, month: 1 })
expect(isAfter({ year: 2027, month: 1 }, { year: 2026, month: 12 })).toBe(true)
expect(isAfter({ year: 2026, month: 12 }, { year: 2026, month: 12 })).toBe(false)
const t = [{ id: 1, name: "Rent", amount: 800000, active: true }, { id: 2, name: "Old gym", amount: 90000, active: false }]
expect(buildNewMonth(3000000, t)).toEqual({ income: 3000000, fixedCosts: [{ templateId: 1, name: "Rent", amount: 800000 }] })
expect(buildNewMonth(null, []).income).toBe(0)
```
- [ ] **Step 2:** `npm test`. Expected: FAIL, functions not found.
- [ ] **Step 3:** Write the functions. `parseAmount`: strip commas and spaces, match `/^\d+(\.\d{1,2})?$/`, convert without floats (split on `.`), reject 0 and anything over `MAX_AMOUNT`. `currentYearMonth`: `Intl.DateTimeFormat("en-CA", { timeZone, year: "numeric", month: "numeric" }).formatToParts(now)`. Shortcut comment in `money.ts`: `// shortcut: assumes 2 decimal places; change if APP_CURRENCY is JPY-like`.
- [ ] **Step 4:** `npm test`. Expected: all pass.
- [ ] **Step 5:** Commit `feat: money and month logic`.

---

### Lesson 2: Home page with fake data

**You learn:** server components, layouts, components with props, and mobile-first Tailwind.

**Files:** Create `src/components/LeftSummary.tsx`, `src/components/ExpenseList.tsx`; Modify `src/app/page.tsx`, `src/app/layout.tsx`

**Interfaces (produces):**
```ts
LeftSummary({ income, fixedTotal, spent, currency }: { income: number; fixedTotal: number; spent: number; currency: string })
ExpenseList({ expenses, currency }: { expenses: { id: number; date: string; amount: number; note: string | null }[]; currency: string })
```

- [ ] **Step 1:** Build `LeftSummary`. It shows a big "Left" number (via `calcLeft` and `formatMoney`) in red when negative, then rows for Income, Fixed and Spent. Match the phone sketch in the spec.
- [ ] **Step 2:** Build `ExpenseList`: newest first, date + note + amount; empty state "No spending yet this month."
- [ ] **Step 3:** Render both on `src/app/page.tsx` with hard-coded data, including one case where Left is negative.
- [ ] **Step 4:** Check at phone width (browser devtools, 375px): no sideways scroll, Left readable at a glance.
- [ ] **Step 5:** Commit `feat: home page layout with sample data`.

---

### Lesson 3: Database with Prisma and Neon

**You learn:** env variables, the Prisma schema, migrations, and why the Prisma client is a singleton in dev.

**Files:** Create `prisma/schema.prisma`, `src/lib/db.ts`, `.env` (git-ignored), `.env.example`

- [ ] **Step 1:** Create a free Neon project (neon.tech). Put its connection string in `.env` as `DATABASE_URL`. Add `.env.example` listing `DATABASE_URL`, `APP_CURRENCY`, `APP_TIMEZONE`, `ALLOWED_EMAIL`, `AUTH_SECRET`, `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET` with no values. Check that `.env` is in `.gitignore`.
- [ ] **Step 2:** `npm install prisma @prisma/client` then `npx prisma init`. Follow what `init` prints for the current Prisma version (generator / output settings).
- [ ] **Step 3:** Write the models:
```prisma
model FixedCostTemplate {
  id        Int     @id @default(autoincrement())
  name      String
  amount    Int
  active    Boolean @default(true)
  sortOrder Int     @default(0)
  monthCopies MonthFixedCost[]
}
model Month {
  id         Int   @id @default(autoincrement())
  year       Int
  month      Int
  income     Int   @default(0)
  fixedCosts MonthFixedCost[]
  expenses   Expense[]
  @@unique([year, month])
}
model MonthFixedCost {
  id         Int    @id @default(autoincrement())
  monthId    Int
  month      Month  @relation(fields: [monthId], references: [id], onDelete: Cascade)
  templateId Int?
  template   FixedCostTemplate? @relation(fields: [templateId], references: [id], onDelete: SetNull)
  name       String
  amount     Int
}
model Expense {
  id        Int      @id @default(autoincrement())
  monthId   Int
  month     Month    @relation(fields: [monthId], references: [id], onDelete: Cascade)
  date      DateTime @db.Date
  amount    Int
  note      String?
  createdAt DateTime @default(now())
}
```
- [ ] **Step 4:** `npx prisma migrate dev --name init`. Expected: migration created and applied; `npx prisma studio` shows 4 empty tables.
- [ ] **Step 5:** Write `src/lib/db.ts`, exporting `prisma`, using the `globalThis` singleton pattern from the Prisma Next.js docs.
- [ ] **Step 6:** Commit `feat: prisma schema and db client`. The migration folder is included; `.env` is not.

---

### Lesson 4: Real months from the database

**You learn:** dynamic routes (`[year]/[month]`), async server components, `redirect` and `notFound`, and transactions.

**Files:** Create `src/lib/months.ts`, `src/app/(app)/m/[year]/[month]/page.tsx`, `src/components/MonthNav.tsx`; Modify `src/app/page.tsx`

**Interfaces:**
- Consumes: `buildNewMonth`, `currentYearMonth`, `prevYearMonth`, `nextYearMonth`, `isAfter`, `calcLeft`, `prisma`
- Produces:
```ts
export async function getOrCreateMonth(ym: YearMonth): Promise<Month & { fixedCosts: MonthFixedCost[]; expenses: Expense[] }>
MonthNav({ ym, current }: { ym: YearMonth; current: YearMonth })   // › hidden when ym == current
```

- [ ] **Step 1:** Write `getOrCreateMonth`. Find by `year_month`. If it's missing, read the previous month's income and the active templates, call `buildNewMonth`, and create the Month with nested `fixedCosts` in one `create`. Include `fixedCosts` and `expenses` (expenses ordered by `date desc, createdAt desc`).
- [ ] **Step 2: Race handling.** Wrap the create in try/catch. On Prisma error code `P2002` (unique violation, which means another request created it first), fetch again and return that row.
- [ ] **Step 3:** In `src/app/page.tsx`, `redirect` to `/m/{year}/{month}` for `currentYearMonth(process.env.APP_TIMEZONE!)`.
- [ ] **Step 4:** In the month page, parse params. Call `notFound()` if they're not integers, if the month is outside 1–12, or if `isAfter(ym, current)`. Then render `MonthNav`, `LeftSummary`, `ExpenseList` from real data.
- [ ] **Step 5:** Check: `/` lands on this month. ‹ goes back. › hides on the current month. `/m/2099/1` gives 404. Prisma Studio shows one Month row per month visited.
- [ ] **Step 6:** Commit `feat: months load from database`.

---

### Lesson 5: Logging spending with server actions

**You learn:** server actions, `<form action>`, zod, `useActionState` for errors, `useFormStatus` for pending state, and `revalidatePath`.

**Files:** Create `src/lib/validation.ts`, `src/app/actions/expenses.ts`, `src/components/ExpenseForm.tsx`, `src/components/SubmitButton.tsx`; Test `tests/validation.test.ts`; Modify `ExpenseList.tsx` (edit/delete)

**Interfaces (produces):**
```ts
export const NOTE_MAX = 200
export function expenseSchema(ym: YearMonth): z.ZodType<{ amount: number; note: string | null; date: string }>  // date "YYYY-MM-DD"
export type ActionState = { error?: string; ok?: boolean }   // in validation.ts, shared by all actions
export async function addExpense(prev: ActionState, formData: FormData): Promise<ActionState>     // fields: year, month, amount, note, date
export async function updateExpense(prev: ActionState, formData: FormData): Promise<ActionState>  // + id
export async function deleteExpense(formData: FormData): Promise<void>                            // id
SubmitButton({ children }: { children: React.ReactNode })  // disabled while pending
```

- [ ] **Step 1: Write the failing tests** in `tests/validation.test.ts`:
```ts
const s = expenseSchema({ year: 2026, month: 11 })
expect(s.parse({ amount: "120.50", note: "Lunch", date: "2026-11-20" })).toEqual({ amount: 12050, note: "Lunch", date: "2026-11-20" })
expect(s.parse({ amount: "85", note: "", date: "2026-11-01" }).note).toBeNull()
expect(s.safeParse({ amount: "0", note: "", date: "2026-11-20" }).success).toBe(false)
expect(s.safeParse({ amount: "50", note: "", date: "2026-12-01" }).success).toBe(false)   // other month
expect(s.safeParse({ amount: "50", note: "x".repeat(201), date: "2026-11-20" }).success).toBe(false)
```
- [ ] **Step 2:** `npm test`. Expected: FAIL.
- [ ] **Step 3:** Write `expenseSchema`: amount goes through `parseAmount` (refine non-null), note is trimmed and empty becomes null, date must start with `YYYY-MM-`.
- [ ] **Step 4:** `npm test`. Expected: pass.
- [ ] **Step 5:** Write the three actions (`"use server"`). Validate, call `getOrCreateMonth` for the month, write with prisma, `revalidatePath` the month page, and return `{ error }` on bad input.
- [ ] **Step 6:** Build `ExpenseForm` (client component). Amount uses `inputMode="decimal"` so phones show a number pad. Note is optional. Date defaults to today, or the 1st for past months. Show the error from `useActionState`. Use `SubmitButton`. Add edit (inline form) and delete (form with confirm) to `ExpenseList`.
- [ ] **Step 7:** Check on a phone-width screen: add "120.5 Lunch" and Left drops by 120.50. Add "abc" and an error shows. Double-tap Add and only one entry appears. Edit the date to next month and it's rejected. Delete works after confirming.
- [ ] **Step 8:** Commit `feat: add, edit, delete expenses`.

---

### Lesson 6: Income and fixed costs

**You learn:** nested routes, more server actions, checkboxes in forms, and reusing schemas.

**Files:** Create `src/app/(app)/m/[year]/[month]/setup/page.tsx`, `src/app/(app)/fixed-costs/page.tsx`, `src/app/actions/month.ts`, `src/app/actions/templates.ts`; Modify `src/lib/validation.ts`, `tests/validation.test.ts`

**Interfaces (produces):**
```ts
export const incomeSchema: z.ZodType<{ income: number }>            // "0" allowed here; uses parseAmount or "0"
export const fixedCostSchema: z.ZodType<{ name: string; amount: number }>  // name 1-60 chars
export async function updateIncome(prev: ActionState, fd: FormData): Promise<ActionState>          // year, month, income
export async function saveMonthFixedCost(prev: ActionState, fd: FormData): Promise<ActionState>    // year, month, id?, name, amount
export async function deleteMonthFixedCost(fd: FormData): Promise<void>                            // id
export async function saveTemplate(prev: ActionState, fd: FormData): Promise<ActionState>          // id?, name, amount, applyToCurrent ("on" | null)
export async function deactivateTemplate(fd: FormData): Promise<void>                              // id
```

- [ ] **Step 1: Failing tests:** `incomeSchema.parse({ income: "0" }).income === 0`; `incomeSchema.parse({ income: "30,000" }).income === 3000000`; `fixedCostSchema.safeParse({ name: "", amount: "100" }).success === false`; `fixedCostSchema.safeParse({ name: "x".repeat(61), amount: "100" }).success === false`.
- [ ] **Step 2:** `npm test`, see it fail. Write the schemas. `npm test`, see it pass.
- [ ] **Step 3:** Setup page: an income form and this month's fixed-cost list (add/edit/delete), all using `saveMonthFixedCost` / `deleteMonthFixedCost` / `updateIncome`. Link to it from the month page.
- [ ] **Step 4:** Fixed-costs page: list active templates, plus an add/edit form with the checkbox "also apply to this month" (checked by default). In `saveTemplate`, if the box is checked, update or create the current month's `MonthFixedCost` with that `templateId`. Never touch other months. "Remove" calls `deactivateTemplate` (sets `active=false`, keeps history).
- [ ] **Step 5:** Check: add template "Netflix 419" with the box ticked, and this month's Left drops by 419. Edit the template amount with the box unticked, and this month stays the same. Open a past month and its fixed costs are unchanged.
- [ ] **Step 6:** Commit `feat: income and fixed costs`.

---

### Lesson 7: History and export

**You learn:** Prisma aggregation, list pages, and route handlers (`route.ts`) that return a file.

**Files:** Create `src/app/(app)/history/page.tsx`, `src/app/api/export/route.ts`

- [ ] **Step 1:** History page: every Month row, newest first, showing income / fixed / spent / left (`calcLeft`). Each row links to `/m/{y}/{m}`. Empty state when there's only the current month.
- [ ] **Step 2:** `GET /api/export`: return all templates, months, month fixed costs and expenses as JSON, with header `Content-Disposition: attachment; filename="money-journal-YYYY-MM-DD.json"`.
- [ ] **Step 3:** Add a "Download my data" link on History. Check: the file downloads and opens as valid JSON with your entries.
- [ ] **Step 4:** Commit `feat: history and data export`.

---

### Lesson 8: Sign in with Google, owner only

**You learn:** Auth.js, OAuth, sessions, route groups, and why a layout check does **not** protect server actions.

**Files:** Create `src/auth.ts`, `src/lib/auth-rules.ts`, `src/app/api/auth/[...nextauth]/route.ts`, `src/app/signin/page.tsx`, `src/app/(app)/layout.tsx`; Test `tests/auth-rules.test.ts`; Modify every file in `src/app/actions/`, `src/app/api/export/route.ts`

**Interfaces (produces):**
```ts
export function isAllowedEmail(email: string | null | undefined, allowed: string | undefined): boolean
export async function requireOwner(): Promise<void>   // redirects to /signin when no valid session
```

- [ ] **Step 1: Failing tests:**
```ts
expect(isAllowedEmail("Me@Gmail.com", "me@gmail.com")).toBe(true)
expect(isAllowedEmail(" me@gmail.com ", "me@gmail.com")).toBe(true)
expect(isAllowedEmail("other@gmail.com", "me@gmail.com")).toBe(false)
expect(isAllowedEmail(null, "me@gmail.com")).toBe(false)
expect(isAllowedEmail("me@gmail.com", undefined)).toBe(false)   // env missing means nobody gets in
expect(isAllowedEmail("me@gmail.com", "")).toBe(false)
```
- [ ] **Step 2:** `npm test`, see it fail. Write `isAllowedEmail` (trim + lowercase, false if either side is empty). `npm test`, see it pass.
- [ ] **Step 3:** In Google Cloud Console, create an OAuth client (web). Set the redirect URI to `http://localhost:3000/api/auth/callback/google`. Put the ID and secret in `.env`, and run `npx auth secret` for `AUTH_SECRET`. Check authjs.dev for the current v5 install command.
- [ ] **Step 4:** Write `src/auth.ts` with the Google provider and a `signIn` callback that returns `isAllowedEmail(profile.email, process.env.ALLOWED_EMAIL)`. Export `auth`, `signIn`, `signOut`, `handlers`, `requireOwner`. Wire up the route handler. Build `/signin` with a form whose server action calls `signIn("google")`.
- [ ] **Step 5:** Move the month/setup/fixed-costs/history pages under `src/app/(app)/` (already there if you followed the paths). In its `layout.tsx`, call `await requireOwner()`, then render the nav (This month · Fixed costs · History · Sign out).
- [ ] **Step 6:** Add `await requireOwner()` as the first line of **every** server action and of `GET /api/export`. Teaching point: actions are public POST endpoints, and the layout doesn't run for them.
- [ ] **Step 7:** Check: in a private window, `/` goes to /signin. Your Google account gets in. A different Google account is refused. `/api/export` with no session goes to /signin.
- [ ] **Step 8:** Commit `feat: owner-only Google sign-in`.

---

### Lesson 9: Deploy to Vercel

**You learn:** GitHub → Vercel deploys, production env vars, and running migrations on deploy.

**Files:** Modify `package.json` (build script), `README.md`

- [ ] **Step 1:** `gh auth status`. Expected: the active account is **Namostpc**. If not, `gh auth switch --user Namostpc`. Then `gh repo create journal-projects --private --source . --push`.
- [ ] **Step 2:** Set the build script to `prisma generate && prisma migrate deploy && next build`.
- [ ] **Step 3:** Import the repo in Vercel. Add all env vars from `.env.example`, with production values. Use the Neon connection string for production (the same DB or a separate Neon branch, your call).
- [ ] **Step 4:** In Google Cloud Console, add the redirect URI `https://<your-app>.vercel.app/api/auth/callback/google`.
- [ ] **Step 5:** Write `README.md`: what the app is, the env vars, local setup (`npm install`, `.env`, `npx prisma migrate dev`, `npm run dev`), and the deploy notes above.
- [ ] **Step 6: Smoke test on your phone:** open the Vercel URL, sign in, add an expense, and see Left drop. Open it on the Mac and see the same entry.
- [ ] **Step 7:** Commit `docs: readme and deploy`, then `git push`.
