# Brewbook — Coffee Shop POS & Operations

A single-machine, offline-first POS and back-office system for independent coffee shops and small chains. Built with Next.js, React, and SQLite. Designed so a future upgrade to multi-terminal LAN deployment requires only swapping the database connection — no application rewrite.

This document is the full build specification. A working HTML prototype (`brewbook.html`) accompanies this spec as a reference implementation. The behavior in the prototype is the source of truth for business logic; this spec is the source of truth for architecture, file structure, and conventions.

---

## 1. Goals & Non-Goals

**Goals**
- Runs offline. No internet required after install. No external API dependencies.
- Single-machine deployment first. Data lives in a local SQLite file the user can back up by copying.
- Architected so that swapping SQLite for a networked database (Postgres or shared SQLite over LAN) is a single configuration change — the application code does not change.
- Recipe-driven inventory: every product can have a Bill of Materials. Selling a product automatically deducts raw materials and computes COGS.
- Real-time double-entry bookkeeping. P&L and Balance Sheet are always live, computed from the journal.
- Multi-currency, configurable VAT, English + Arabic UI (RTL).
- Production-grade: typed end-to-end, tested, audit-logged, with a clean migration path.

**Non-goals (for v1)**
- No cloud sync, no multi-shop consolidation, no online ordering.
- No payment-processor integrations (Stripe, etc.). Payment methods are recorded as labels only.
- No employee scheduling or payroll calculation. Just timesheet logging.
- No customer-facing display or kitchen display. Single-screen POS only.

---

## 2. Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Framework | Next.js 15 (App Router) | Server actions for write paths, RSC for read paths, single deployable unit. |
| UI | React 19 + Tailwind CSS 4 + shadcn/ui | Standard modern stack, Claude Code knows it well. |
| Language | TypeScript (strict) | Catches the bookkeeping bugs that JS would let through. |
| Database | SQLite via `better-sqlite3` | Synchronous, fast, single-file. Wrapped in Drizzle ORM. |
| ORM | Drizzle ORM | Type-safe, SQL-like, easy to read. Migration support built-in. |
| State (client) | TanStack Query | For cached reads + optimistic writes. No Redux. |
| Forms | React Hook Form + Zod | Validation lives in `lib/schemas.ts` and is shared client/server. |
| Date/time | `date-fns` | Lightweight, immutable. Always store ISO strings, format at the edge. |
| Money | Plain `number` representing **major units** (e.g. 12.50 = 12 pounds 50 piastres). All money operations go through `lib/money.ts` helpers (`add`, `mul`, `round`). Round to currency decimals only at display/persistence boundary. |
| i18n | `next-intl` | Server + client, RTL support. |
| Charts | `recharts` | Used in dashboard. |
| PDF/Print | Browser print CSS for receipts. `@react-pdf/renderer` for reports if needed. |
| Testing | Vitest (unit) + Playwright (e2e) | Vitest for accounting/inventory logic, Playwright for the cash-flow happy path. |
| Lint/format | ESLint + Prettier, default Next.js config | |
| Package manager | pnpm | |

---

## 3. Project Structure

```
brewbook/
├── app/
│   ├── (pos)/                       # POS layout (no sidebar — full-screen)
│   │   └── pos/page.tsx
│   ├── (admin)/                     # Admin layout (sidebar)
│   │   ├── layout.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── orders/page.tsx
│   │   ├── products/page.tsx
│   │   ├── recipes/page.tsx
│   │   ├── inventory/page.tsx
│   │   ├── purchases/page.tsx
│   │   ├── expenses/page.tsx
│   │   ├── reports/
│   │   │   ├── pnl/page.tsx
│   │   │   └── balance-sheet/page.tsx
│   │   └── settings/page.tsx
│   ├── api/                          # Only used for things server actions can't do (file download/upload)
│   │   ├── backup/route.ts
│   │   └── restore/route.ts
│   ├── layout.tsx
│   └── globals.css
│
├── server/                            # Server-only code. NEVER imported from client components.
│   ├── db/
│   │   ├── client.ts                  # `db` instance (better-sqlite3 + drizzle)
│   │   ├── schema.ts                  # All tables in one file. Single source of truth.
│   │   ├── migrations/                # Drizzle-generated migrations. Committed.
│   │   └── seed.ts                    # Sample data loader.
│   ├── repos/                         # Data access: one file per aggregate.
│   │   ├── products.ts
│   │   ├── materials.ts
│   │   ├── orders.ts
│   │   ├── purchases.ts
│   │   ├── expenses.ts
│   │   └── journal.ts
│   ├── services/                      # Business logic. Pure-ish functions, no HTTP.
│   │   ├── posting.ts                 # postJournal, reverseJournal
│   │   ├── inventory.ts               # consumeRecipe, recordStockMove, weightedAvgCost
│   │   ├── orders.ts                  # finalizeOrder
│   │   ├── purchases.ts               # finalizePurchase
│   │   ├── expenses.ts                # finalizeExpense
│   │   ├── reports.ts                 # getPnL, getBalanceSheet, getAccountBalance
│   │   └── audit.ts                   # logAction
│   └── actions/                       # Server actions called from forms/buttons.
│       ├── orders.ts                  # placeOrderAction, voidOrderAction
│       ├── products.ts
│       ├── materials.ts
│       └── ... etc
│
├── components/                        # React components. Default export for routes, named for reusables.
│   ├── ui/                            # shadcn/ui primitives (button, input, dialog, etc.)
│   ├── pos/                           # POS-specific: ProductGrid, Cart, PaymentDialog, ReceiptDialog
│   ├── admin/                         # Admin: DataTable, FormShell, StatTile, FinReportRow
│   └── shared/                        # Money display, DateDisplay, LangSwitch, CurrencySelect
│
├── lib/                               # Isomorphic helpers. Used on both server and client.
│   ├── money.ts                       # add, sub, mul, div, round, format
│   ├── units.ts                       # convertQty (g↔kg, ml↔l, pc)
│   ├── currencies.ts                  # CURRENCIES table (12 entries, see §6.2)
│   ├── accounts.ts                    # Chart of accounts as a TS const (see §5.1)
│   ├── schemas.ts                     # Zod schemas for every form payload
│   └── i18n/
│       ├── en.json
│       └── ar.json
│
├── tests/
│   ├── unit/
│   │   ├── posting.test.ts            # journal balances, contra accounts
│   │   ├── inventory.test.ts          # weighted avg, unit conversion
│   │   ├── reports.test.ts            # P&L matches manual calc
│   │   └── orders.test.ts             # finalizeOrder happy path + edge cases
│   └── e2e/
│       └── happy-path.spec.ts         # Open POS → add 2 cappuccinos → pay → check stock → check P&L
│
├── public/
├── drizzle.config.ts
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── package.json
├── README.md                          # Install + run instructions, in this order:
│                                      #   1. Install Node 20+, 2. pnpm install, 3. pnpm db:migrate,
│                                      #   4. pnpm db:seed, 5. pnpm dev, 6. open http://localhost:3000
├── CLAUDE.md                          # Rules for AI sessions (see §13)
└── SPEC.md                            # This file.
```

**Layering rule.** Routes call actions. Actions call services. Services call repos. Repos call `db`. Components never import from `server/`. The `lib/` layer is the only thing both sides may import.

---

## 4. Database Schema

All tables share these conventions:
- `id` is `text primary key`, default value: a generated CUID2 (use the `@paralleldrive/cuid2` package).
- All money columns are `real` (SQLite has no decimal type; we use `number` consistently and round at the edge).
- All timestamps are `text not null default (current_timestamp)`, ISO format. Never store local time.
- Soft delete: tables that need it have `deleted_at text`. The repo layer filters on this by default.
- Every table has `created_at` and `updated_at`. The repos write `updated_at` on every put.

**Schema file:** `server/db/schema.ts`. Drizzle.

### 4.1 Tables

```ts
// settings — single row, id always 'app'
settings { id, currency, vat_pct, shop_name, address, phone, tax_id,
           receipt_footer, language, opening_cash, initial_capital,
           created_at, updated_at }

// categories — for the POS product grid
categories { id, name, sort_order, created_at, updated_at }

// products — sold through POS
products { id, sku unique, name, category_id fk→categories.id,
           price, taxable, active, created_at, updated_at, deleted_at }

// recipe_items — Bill of Materials (junction)
recipe_items { id, product_id fk→products.id, material_id fk→materials.id,
               qty, unit, sort_order }

// materials — raw inventory
materials { id, sku unique, name, unit, cost_per_unit, stock,
            reorder_point, supplier, created_at, updated_at, deleted_at }

// suppliers — optional, free-text supplier name on materials is also fine
suppliers { id, name, phone, notes, created_at, updated_at }

// purchases — goods receipts
purchases { id, date, supplier_id, total, paid, status,
            created_at, updated_at }
purchase_items { id, purchase_id fk, material_id fk, qty, unit_cost }

// expenses
expenses { id, date, category, vendor, amount, notes,
           paid_from_account, created_at, updated_at }

// orders — POS sales
orders { id, num integer unique, date, mode, subtotal, discount, vat, total,
         cogs, payment_method, payment_amount, payment_change,
         cashier_id, status,    -- 'paid' | 'voided'
         voided_at, voided_by, voided_reason,
         created_at, updated_at }
order_items { id, order_id fk, product_id fk, name, price, qty,
              line_total, taxable }

// stock_moves — every inventory change. Append-only.
stock_moves { id, date, material_id fk, qty (positive),
              type ('in'|'out'|'adjust'|'waste'),
              ref_type, ref_id, cost_per_unit, signed_delta,
              created_at }

// journal — double-entry bookkeeping. Append-only.
journal { id, date, ref_type, ref_id, memo, reversed (bool),
          created_at }
journal_lines { id, journal_id fk, account text, dr real, cr real, memo }
-- CHECK (dr >= 0 AND cr >= 0 AND (dr = 0 OR cr = 0))
-- per-line, only one of dr/cr is non-zero. Trigger or repo enforces sum(dr) = sum(cr) per journal.

// users — staff
users { id, name, role ('owner'|'manager'|'cashier'),
        pin (hashed), active, created_at, updated_at, deleted_at }

// shifts — cashier sessions, optional in v1
shifts { id, opened_at, closed_at, opened_by fk→users.id,
         opening_float, closing_cash, expected_cash, variance, notes }

// audit_log — every write action. Append-only.
audit_log { id, ts, user_id, action, entity, entity_id, before_json, after_json, ip }
```

### 4.2 Indexes

```sql
create index idx_orders_date on orders(date);
create index idx_orders_num on orders(num);
create index idx_journal_lines_account on journal_lines(account);
create index idx_journal_date on journal(date);
create index idx_stock_moves_material on stock_moves(material_id);
create index idx_stock_moves_date on stock_moves(date);
create index idx_recipe_product on recipe_items(product_id);
```

### 4.3 Migrations

Drizzle-kit generates migrations under `server/db/migrations/`. Commit them. Apply with `pnpm db:migrate`. Never edit a migration after it's been applied; always add a new one.

---

## 5. Accounting Engine

### 5.1 Chart of Accounts

`lib/accounts.ts` exports a `const` array. Account ids are stable strings — they appear in journal lines forever. Don't renumber.

```ts
[
  // Assets (debit-normal)
  { id: '1010', name: 'Cash on Hand',                type: 'asset' },
  { id: '1020', name: 'Bank Account',                type: 'asset' },
  { id: '1030', name: 'Accounts Receivable',         type: 'asset' },
  { id: '1100', name: 'Inventory — Raw Materials',   type: 'asset' },
  { id: '1500', name: 'Equipment & Fixtures',        type: 'asset' },
  { id: '1510', name: 'Accumulated Depreciation',    type: 'asset', contra: true },

  // Liabilities (credit-normal)
  { id: '2010', name: 'Accounts Payable',            type: 'liability' },
  { id: '2020', name: 'VAT Payable',                 type: 'liability' },
  { id: '2030', name: 'Accrued Expenses',            type: 'liability' },
  { id: '2100', name: 'Loans Payable',               type: 'liability' },

  // Equity (credit-normal)
  { id: '3010', name: "Owner's Capital",             type: 'equity' },
  { id: '3020', name: 'Retained Earnings',           type: 'equity' },

  // Revenue (credit-normal)
  { id: '4010', name: 'Sales Revenue',               type: 'revenue' },
  { id: '4020', name: 'Discounts Given',             type: 'revenue', contra: true },

  // Expenses (debit-normal)
  { id: '5010', name: 'Cost of Goods Sold',          type: 'expense' },
  { id: '5100', name: 'Wastage & Spoilage',          type: 'expense' },
  { id: '6010', name: 'Rent',                        type: 'expense' },
  { id: '6020', name: 'Salaries & Wages',            type: 'expense' },
  { id: '6030', name: 'Utilities',                   type: 'expense' },
  { id: '6040', name: 'Marketing',                   type: 'expense' },
  { id: '6050', name: 'Supplies',                    type: 'expense' },
  { id: '6060', name: 'Maintenance',                 type: 'expense' },
  { id: '6070', name: 'Depreciation',                type: 'expense' },
  { id: '6080', name: 'Other Expenses',              type: 'expense' },
]
```

Type = `'asset' | 'liability' | 'equity' | 'revenue' | 'expense'`. Optional `contra: true` flips the sign.

### 5.2 Journal Posting (`server/services/posting.ts`)

```ts
postJournal(entry: {
  date?: string;            // defaults to now
  refType: string;          // 'order' | 'purchase' | 'expense' | 'opening' | 'adjustment' | ...
  refId: string;
  memo?: string;
  lines: Array<{
    account: string;        // account id from chart of accounts
    dr?: number;            // never negative
    cr?: number;            // never negative
    memo?: string;
  }>
}): Promise<JournalEntry>
```

**Invariants enforced before writing**, throw on violation:
1. `lines.length >= 2`
2. For each line: exactly one of `dr` or `cr` is positive; the other is 0 or omitted.
3. `sum(dr)` equals `sum(cr)` within 0.01 (allow for floating-point drift; round both sides to 2 decimals before compare).
4. Every `account` string exists in the chart.

The function writes to `journal` and `journal_lines` in **one SQLite transaction**.

```ts
reverseJournal(refType: string, refId: string): Promise<void>
```
Find every non-reversed entry for that ref, write a mirror entry (swap dr/cr) tagged `refType: 'reverse_<original>'`, mark the original `reversed = true`. Used for voids and corrections.

### 5.3 Account Balance

```ts
getAccountBalance(accountId: string, asOf?: string): Promise<number>
```

Sum `dr` and `cr` from `journal_lines` joined with `journal` filtered by `date <= asOf` if given. For debit-normal accounts return `dr - cr`; for credit-normal return `cr - dr`. Flip sign for contra accounts. **Implement as a single SQL query**, not in JS.

### 5.4 P&L

```ts
getPnL(from?: string, to?: string): Promise<{
  revenue: Record<string, number>;        // by account id
  expense: Record<string, number>;
  totalRevenue: number;
  totalCogs: number;                      // sum of accounts 5010 + 5100
  totalOpex: number;                      // all other expense accounts
  grossProfit: number;                    // totalRevenue - totalCogs
  netProfit: number;                      // grossProfit - totalOpex
}>
```

Single SQL query grouped by account. Then aggregate in JS.

### 5.5 Balance Sheet

```ts
getBalanceSheet(asOf?: string): Promise<{
  asset:     Array<{ id, name, balance }>;
  liability: Array<{ id, name, balance }>;
  equity:    Array<{ id, name, balance }>;
  totalAssets, totalLiabilities, totalEquity: number;
}>
```

Build by calling `getAccountBalance` for every non-revenue, non-expense account. Compute Retained Earnings on the fly: `totalNetProfit(from: null, to: asOf)` and add to the equity section as account `3020`. Suppress accounts with zero balance except `1010` and `3020` which always show.

**Self-check**: `Math.abs(totalAssets - (totalLiabilities + totalEquity)) < 0.01`. Surface a visible warning in the UI when not.

---

## 6. Inventory Engine

### 6.1 Unit Conversion (`lib/units.ts`)

```ts
const UNIT_CONV = {
  // mass — base: g
  g:  { base: 'g',  factor: 1 },
  kg: { base: 'g',  factor: 1000 },
  // volume — base: ml
  ml: { base: 'ml', factor: 1 },
  l:  { base: 'ml', factor: 1000 },
  // count — base: pc
  pc: { base: 'pc', factor: 1 },
};

convertQty(qty: number, fromUnit: string, toUnit: string): number | null
// Returns null when units are incompatible (e.g. g → ml).
// Caller must handle null and surface "incompatible units" errors.
```

Recipe units and material units do not have to match. The conversion happens at consumption time. Only base must match (mass↔mass, volume↔volume, count↔count).

### 6.2 Currencies (`lib/currencies.ts`)

```ts
{
  EGP: { symbol: 'ج.م', name: 'Egyptian Pound', decimals: 2 },
  SAR: { symbol: 'ر.س', name: 'Saudi Riyal',     decimals: 2 },
  AED: { symbol: 'د.إ', name: 'UAE Dirham',      decimals: 2 },
  USD: { symbol: '$',   name: 'US Dollar',       decimals: 2 },
  EUR: { symbol: '€',   name: 'Euro',            decimals: 2 },
  GBP: { symbol: '£',   name: 'British Pound',   decimals: 2 },
  KWD: { symbol: 'د.ك', name: 'Kuwaiti Dinar',   decimals: 3 },
  QAR: { symbol: 'ر.ق', name: 'Qatari Riyal',    decimals: 2 },
  BHD: { symbol: 'د.ب', name: 'Bahraini Dinar',  decimals: 3 },
  OMR: { symbol: 'ر.ع', name: 'Omani Rial',      decimals: 3 },
  JOD: { symbol: 'د.أ', name: 'Jordanian Dinar', decimals: 3 },
  TRY: { symbol: '₺',   name: 'Turkish Lira',    decimals: 2 },
}
```

`Money.format(amount, currencyCode)` reads `decimals` and produces e.g. `"125.40 ج.م"` or `"125.400 د.ك"`.

### 6.3 Stock Movement (`server/services/inventory.ts`)

```ts
recordStockMove(input: {
  materialId: string;
  qty: number;                            // always positive
  type: 'in' | 'out' | 'adjust' | 'waste';
  refType: string;                        // 'order' | 'purchase' | 'adjustment' | 'opening'
  refId: string;
  costPerUnit?: number;                   // required for 'in', else falls back to material.cost_per_unit
}): Promise<StockMove>
```

Computes signed delta: `+qty` for `in` and `adjust+`, `-qty` for `out` / `waste` / `adjust-`. Updates `materials.stock` in the same transaction as the `stock_moves` insert.

```ts
consumeRecipeForOrder(orderId: string): Promise<number>   // returns total COGS
```

For each order line:
1. Load the product's `recipe_items`.
2. For each ingredient, `convertQty(ingredient.qty, ingredient.unit, material.unit)`.
3. Multiply by line `qty`.
4. `recordStockMove(..., type: 'out', refType: 'order', refId: orderId, costPerUnit: material.cost_per_unit)`.
5. Accumulate `consumedQty * cost_per_unit` into COGS.

### 6.4 Weighted-Average Cost on Purchase

When a purchase line increases stock:
```
oldVal      = stock * cost_per_unit
addedVal    = qty * unit_cost
newStock    = stock + qty
new_cost    = newStock > 0 ? (oldVal + addedVal) / newStock : unit_cost
```
Update `materials.stock` and `materials.cost_per_unit` atomically.

### 6.5 Stock-Sufficiency Check (Client + Server)

`canMake(productId, qty): Promise<boolean>` — used by the POS to block adding products to the cart when any recipe ingredient has insufficient stock. Implemented on the server, called from a server action; the result is also cached client-side via TanStack Query so the product grid greys out unavailable items.

Server-side `finalizeOrder` re-checks before committing — never trust the client cache for the final guard.

---

## 7. Order Lifecycle (`server/services/orders.ts`)

### 7.1 `finalizeOrder`

Single SQLite transaction. Order of operations:

```ts
finalizeOrder(input: {
  mode: 'dine-in' | 'takeaway' | 'delivery';
  items: Array<{ productId: string; qty: number; }>;     // server resolves prices
  discount: { type: 'percent' | 'amount'; value: number };
  payment: { method: 'cash' | 'card' | 'wallet'; amountTendered: number };
  cashierId: string;
}): Promise<Order>
```

Steps:
1. Load product rows. Compute `price`, `lineTotal` per item from the **server-side** product price (do not trust client price).
2. Re-check stock sufficiency for every line. Reject with a typed error if any product can't be made.
3. Compute totals server-side: `subtotal`, `discount`, `vatBase = subtotal − discount`, `vat = vatBase × vatPct/100`, `total = vatBase + vat`. Round each step to currency decimals.
4. Validate `payment.amountTendered >= total` for cash, `=== total` for card/wallet.
5. Generate `num = max(orders.num) + 1` inside the transaction.
6. Insert `orders` and `order_items`.
7. `consumeRecipeForOrder(orderId)` → `cogs`. Update `orders.cogs`.
8. Post sales journal:
   ```
   Dr  1010 Cash (or 1020 Bank)    total
       Cr 4010 Sales Revenue            subtotal
   Dr  4020 Discounts Given        discount     (only if > 0)
       Cr 2020 VAT Payable              vat     (only if > 0)
   ```
9. Post COGS journal (if cogs > 0):
   ```
   Dr  5010 Cost of Goods Sold     cogs
       Cr 1100 Inventory                cogs
   ```
10. Append to `audit_log`.
11. Commit. On any failure, the whole transaction rolls back — including stock changes.

Return the persisted order with all fields populated.

### 7.2 `voidOrder`

```ts
voidOrder(orderId: string, reason: string, userId: string): Promise<void>
```
1. Load order; reject if already voided.
2. `reverseJournal('order', orderId)` and `reverseJournal('order_cogs', orderId)`.
3. For every consumed `stock_moves` of the order, write an opposite move (return to inventory).
4. Set `orders.status = 'voided'`, `voided_at`, `voided_by`, `voided_reason`.
5. Audit-log.

### 7.3 Receipt

Receipts are rendered to a print-friendly route: `/print/receipt/[orderId]`. CSS `@media print` hides everything except the receipt block, sized for thermal printer (80mm). Auto-trigger `window.print()` after open. Same fields as the prototype.

---

## 8. Page-by-Page Spec

For every page below, "data" is fetched in a Server Component using a repo function. Mutations go through Server Actions. Forms use React Hook Form + Zod. Tables use a shared `DataTable` component with built-in column sort, search, pagination.

### 8.1 POS — `/pos`

Layout: full screen, no admin sidebar. Two columns: product browser (left), cart (right).

**Left column**
- Category tabs across the top, horizontally scrollable.
- Product grid below: tiles showing name, price, stock-indicator badge ("Out of stock" red, otherwise nothing). Disabled state on out-of-stock tiles.
- Tap adds to cart. Tap-and-hold opens product detail modal (future; not v1).

**Right column (cart panel)**
- Header: order number ("Order #N+1"), live clock.
- Order-mode pills: Dine-in / Takeaway / Delivery.
- Cart list: each row shows name, line total, qty controls, remove.
- Totals block: subtotal, discount (if any), VAT (with % shown), grand total.
- Action buttons: Discount, Clear, Charge.

**Discount modal**: percent or fixed amount.

**Payment modal**: method dropdown (Cash / Card / Wallet), amount tendered (auto-fill total for non-cash), live change calculation. Confirm button is disabled until tendered ≥ total. On confirm, call `placeOrderAction`, on success show receipt modal and reset cart.

**Empty cart state**: "Tap a product to start an order".

### 8.2 Dashboard — `/dashboard`

Server-rendered. KPIs at top:
- Today's sales (count + amount)
- 7-day sales
- MTD net profit (with margin %)
- Cash on hand, bank balance
- Inventory value, low-stock count

Below, two columns:
- Today's sales by hour — bar chart from `recharts`.
- Top sellers (this month) — top 8 products by revenue, table.

If any low-stock items: yellow alert card listing them.

### 8.3 Orders — `/orders`

Date-range filter (default: today). Table with columns: #, datetime, mode, items, payment method, subtotal, VAT, total, COGS, margin %, action (View, Void).

Stats row above the table: order count, total sales, VAT collected, total COGS, average ticket.

View action opens the receipt modal. Void action prompts for reason, then calls `voidOrderAction`.

### 8.4 Products — `/products`

Table: SKU, name, category, price, recipe cost (computed live), margin %, status, actions (Edit, Recipe).

Edit modal has the basic fields. Recipe action goes to `/recipes/[productId]` or opens the Recipe editor inline (decide based on whichever is simpler — recommend inline modal, large size).

### 8.5 Recipes — `/recipes`

Same as Products but the action is "Edit Recipe" only. Recipe editor:
- Header: product name, sell price, recipe cost (live), food cost % (live).
- Rows: material dropdown (filtered to only show materials whose unit base is compatible with the selected recipe unit), qty input, unit dropdown (only compatible units), remove button.
- "+ Add Ingredient" button.
- Save action persists `recipe_items` (delete + insert in a single transaction).

### 8.6 Inventory — `/inventory`

Tile row: total SKUs, stock value, reorder alert count, out-of-stock count.

Table: SKU, name, supplier, stock, unit, cost/unit, stock value, status (OK / Reorder / Out), Edit.

"Stock Adjustment" button opens a modal: pick material, type (in / out / waste), qty, notes. Posts a journal entry — for waste, debit account 5100; for "out" adjustments, debit 6080; for "in" adjustments, credit 3010 (owner contribution).

Edit material modal: SKU, name, unit, cost per unit (read-only after first purchase), reorder point, supplier. New material has an "opening stock" field that posts an `opening` journal entry.

### 8.7 Purchases — `/purchases`

Table: date, supplier, item count, total, status (paid / unpaid), action (Mark Paid).

New purchase modal: date, supplier name, items (material + qty + unit cost rows, add/remove), live total, "Paid in cash now" checkbox. On save:
- For each item, increase stock + update weighted-avg cost.
- Post journal: Dr 1100 / Cr 1010 (paid) or Cr 2010 (on account).

Mark Paid action: Dr 2010 / Cr 1010.

### 8.8 Expenses — `/expenses`

Table: date, category, vendor, notes, paid from, amount.

New expense modal: date, amount, category (dropdown of expense accounts excluding 5010 and 5100), paid from (Cash 1010 / Bank 1020 / On Account 2010), vendor, notes. Posts: Dr [category] / Cr [paid from].

### 8.9 P&L — `/reports/pnl`

Period selector with presets: Today, Last 7 days, This month (default), YTD, All time, Custom. Render as a formal statement:

```
Brewbook Café
Profit & Loss Statement
Period: 2026-05-01 → 2026-05-09

Revenue
  Sales Revenue                         12,450.00
  (less: Discounts Given)                  −350.00
  Total Revenue                         12,100.00

Cost of Goods Sold
  Cost of Goods Sold                     3,180.00
  Wastage & Spoilage                       120.00
  Total COGS                             3,300.00

Gross Profit                             8,800.00 (72.7%)

Operating Expenses
  Rent                                   2,000.00
  Salaries & Wages                       1,500.00
  Utilities                                400.00
  Total Operating Expenses               3,900.00

Net Profit                               4,900.00 (40.5%)
```

Hide accounts with zero balance.

### 8.10 Balance Sheet — `/reports/balance-sheet`

"As of" date picker (default: today). Render formal statement with three sections (Assets, Liabilities, Equity) and totals. Show a green check "✓ Books balanced" or red warning if A ≠ L+E.

### 8.11 Settings — `/settings`

Two columns. Left: shop info form (name, address, phone, tax ID, currency, VAT %, language, receipt footer). Right: data management (Export, Import, Reset) and an About box describing what the app does.

Export: GET `/api/backup` returns the entire SQLite file as a download (`brewbook-YYYY-MM-DD.db`).

Import: POST `/api/restore` accepts a `.db` upload, validates schema, replaces the live DB.

Reset: prompts twice, drops and re-seeds.

---

## 9. UI / Design System

Colors, fonts, and spacing match the prototype. Codify as Tailwind theme tokens:

```css
--bg: #faf7f2;
--bg-2: #f3ede3;
--ink: #1a1410;
--ink-2: #4a3f35;
--ink-3: #8a7e72;
--line: #e6ddcf;
--paper: #fffdf8;
--accent: #c2410c;     /* burnt orange */
--accent-2: #7c2d12;   /* deep coffee */
--accent-3: #166534;   /* money green */
```

Fonts: `Fraunces` (display, weights 500–700, opsz 144), `Inter` (body), `JetBrains Mono` (numbers in tables and receipts), `Cairo` (Arabic). Load via `next/font`.

Components are shadcn/ui based, themed. Tables: dense rows, uppercase tracking-wide headers, mono text-end for numeric columns. Stat tiles: paper background, 2px accent top-border, mono number in display font. Modals: max-width 600px (lg variant 880px), backdrop blur.

RTL: when language = `ar`, set `<html dir="rtl">` and rely on Tailwind logical properties (`ms-`, `me-`, `ps-`, `pe-`).

---

## 10. Forms & Validation

All forms use Zod schemas in `lib/schemas.ts`. Same schema is used by the React Hook Form resolver on the client and by the Server Action on the server. Don't duplicate.

Example:
```ts
// lib/schemas.ts
export const NewExpenseSchema = z.object({
  date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  amount: z.number().positive(),
  category: z.string().min(1),
  vendor: z.string().optional(),
  notes: z.string().optional(),
  paidFromAccount: z.enum(['1010', '1020', '2010']),
});
export type NewExpense = z.infer<typeof NewExpenseSchema>;

// server/actions/expenses.ts
'use server';
export async function createExpenseAction(input: unknown) {
  const data = NewExpenseSchema.parse(input);
  return finalizeExpense(data);
}
```

---

## 11. Seed Data

`server/db/seed.ts` — runs only when the database is empty. Loads:
- App settings: `currency = EGP`, `vat_pct = 14`, shop name, addresses.
- 1 owner + 1 cashier user.
- 6 categories (Espresso, Brewed Coffee, Cold Drinks, Tea & Other, Pastries, Sandwiches).
- 20 raw materials with Egyptian-market suppliers (Juhayna, Almarai, Cadbury, Domty, Daily Bakery, Lipton, etc.) — see prototype `seedIfEmpty()` for exact list.
- 20 products with full recipes — see prototype.
- Opening capital journal entry (50,000 EGP cash → owner's capital).
- Opening inventory journal entry (sum of all material values → owner's capital).

The seed must produce a Balance Sheet where Assets = Liabilities + Equity exactly.

---

## 12. Tests

**Unit tests** (Vitest, fast):

`tests/unit/posting.test.ts`
- A balanced 2-line journal posts.
- An unbalanced journal throws.
- A line with both dr and cr non-zero throws.
- An unknown account throws.
- `reverseJournal` mirrors lines and marks the original.

`tests/unit/inventory.test.ts`
- `convertQty` handles g↔kg, ml↔l, pc identity.
- Incompatible units return null.
- `recordStockMove` with type 'out' decrements stock.
- Weighted-avg cost matches the formula exactly.

`tests/unit/orders.test.ts`
- Place a 2-cappuccino order: stock decrements correctly (32g beans, 300ml milk, 2 cups, 2 lids, 2 sleeves), COGS matches sum-of-parts, journal balances.
- Place an order that exceeds stock: throws, no rows written.
- Void an order: stock restored, journals reversed, status 'voided'.

`tests/unit/reports.test.ts`
- After seed + 1 order, P&L net profit equals revenue − cogs.
- Balance sheet balances within 0.01.

**E2E test** (Playwright):

`tests/e2e/happy-path.spec.ts`
1. Open `/pos`.
2. Tap Cappuccino twice.
3. Charge as cash, tendered exact total.
4. Confirm receipt opens.
5. Navigate to `/inventory`. Confirm beans and milk decremented.
6. Navigate to `/reports/pnl`. Confirm net profit > 0.
7. Navigate to `/reports/balance-sheet`. Confirm "Books balanced".

---

## 13. CLAUDE.md (project rules for AI sessions)

Place this at the repo root. Claude Code reads it on every session.

```markdown
# Brewbook — AI Session Rules

This is a Next.js 15 + SQLite POS app. Read SPEC.md for the full picture.

## Hard rules
- TypeScript strict. No `any` outside test files. No `// @ts-ignore`.
- Money: never use floats for arithmetic. Always go through `lib/money.ts`.
- Database: never bypass the repo layer from a component or action. UI → action → service → repo → db.
- Server actions only mutate inside a single transaction. If you cross transactions, you lose ACID — don't.
- Every write that affects accounts must post to the journal in the same transaction. No exceptions.
- Never edit applied migrations. Always add new ones.
- Test new business logic before considering a task done. `pnpm test` must stay green.
- shadcn/ui components live in `components/ui` and are owned by us — edit them when you need to, no need to keep them stock.

## Style
- Prefer Server Components. Reach for `'use client'` only for interactive widgets.
- Forms use React Hook Form + Zod. The Zod schema lives in `lib/schemas.ts` and is shared.
- Translations: every user-visible string goes through `next-intl`. No hard-coded English in JSX.

## Nice to have
- Keep imports ordered: external, then internal absolute (`@/...`), then relative.
- Co-locate small helpers with their callers; promote to `lib/` when used by 2+ places.
- Comments explain *why*, not *what*.
```

---

## 14. Migration Path to Multi-Terminal

The point of this architecture: when you outgrow one machine, **only the connection layer changes**. Everything above the repo layer stays the same.

Path A — shared SQLite over LAN (simple)
1. Move `brewbook.db` to a NAS share or a small always-on machine.
2. Set `DATABASE_PATH=//nas/brewbook/brewbook.db` in `.env`.
3. SQLite over a network share works for low write contention (one shop, 2-3 terminals). Risk: file locking quirks on Windows shares.

Path B — Postgres (recommended for chains)
1. Add the `pg` driver alongside `better-sqlite3`. Drizzle has both.
2. Replace `server/db/client.ts` with a Postgres connection.
3. Re-generate migrations against Postgres syntax (Drizzle handles dialect differences).
4. Run `pnpm db:migrate` against the Postgres instance.
5. Each terminal runs the same Next.js app, all pointing at the same DB.

Path C — Full SaaS (multi-shop)
- Add a `shops` scoping column to every table.
- Add tenant middleware that filters every query by `currentShopId`.
- Auth becomes shop-scoped.
- This is a v3 concern; do not pre-build for it.

---

## 15. Build Order for Claude Code

When Claude Code starts work on this repo, build in this sequence. Each step ends with a working, committed slice.

1. **Scaffold.** `pnpm create next-app`, install dependencies, set up Tailwind, shadcn/ui, Drizzle, better-sqlite3, Zod, RHF, TanStack Query, next-intl, date-fns, recharts, Playwright, Vitest. Commit.
2. **Schema + migrations.** Write `server/db/schema.ts` for every table in §4. Generate the first migration. Commit.
3. **Seed.** Write `server/db/seed.ts`. Run it. Inspect the DB with `pnpm drizzle-kit studio`. Commit.
4. **Accounting engine.** Write `server/services/posting.ts` and `reports.ts`. Write unit tests. Make them pass. Commit.
5. **Inventory engine.** Write `server/services/inventory.ts`. Unit tests. Commit.
6. **Order lifecycle.** Write `server/services/orders.ts`. Unit tests, including the void path. Commit.
7. **POS UI.** Build `/pos` end-to-end. The happy-path Playwright test runs green. Commit.
8. **Admin shell.** Layout, sidebar, dashboard. Commit.
9. **Admin pages**, in this order: Orders, Products, Recipes, Inventory, Purchases, Expenses. Commit per page.
10. **Reports.** P&L, then Balance Sheet. Commit.
11. **Settings + backup/restore.** Commit.
12. **i18n.** Add Arabic translations and RTL support. Commit.
13. **Polish.** Print stylesheets, keyboard shortcuts on POS (1-9 = quick-add top products, Enter = pay), low-stock toast on receipt. Commit.

At each step, the app must run and the tests must pass. No long-lived broken branches.

---

## 16. Reference

Open the included `brewbook.html` in a browser to see the working prototype. The visual design, copy, and business logic there are the reference. Anywhere this spec is silent, the prototype's behavior is the answer.

End of spec.
