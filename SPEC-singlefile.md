# Brewbook (Single-File HTML Edition) — Build Spec

A complete coffee shop POS and back-office system delivered as **one self-contained HTML file** (`brewbook.html`). No build step, no Node, no server, no installer. Double-click the file and it runs in any modern browser. All data lives in the browser's IndexedDB.

The architecture is deliberately layered so a future move to a Node + SQLite local server is a single-file swap (the data layer), not an application rewrite.

This spec exists so Claude Code can extend, refactor, or reproduce the app from scratch. A working reference build (`brewbook.html`) accompanies this spec — when this document is silent, that file's behavior is the answer.

---

## 1. Goals & Non-Goals

**Goals**
- Single HTML file. No external runtime dependencies. Works fully offline after first open (Google Fonts is the one network call; if blocked, fall back to system fonts and continue).
- IndexedDB persistence. Data survives reloads, power cuts, and reboots.
- EGP default currency, but the user can switch to any of the 12 supported currencies from Settings without losing data.
- Editable VAT %.
- English + Arabic UI with RTL.
- Recipe-driven inventory: selling a product deducts its raw materials and computes COGS automatically.
- Real-time double-entry bookkeeping. P&L and Balance Sheet always live.
- Backup and restore via JSON export/import.
- The data layer is wrapped in a `db.*` interface. Swapping IndexedDB for a `fetch()`-based client that talks to a Node + SQLite server is a single-file change in §10.

**Non-goals (single-file edition)**
- No multi-terminal real-time sync. (One browser = one set of books. The migration spec addresses this.)
- No payment-processor integration. Methods are labels.
- No build tooling. No bundler, no transpiler, no TypeScript. Plain HTML/CSS/vanilla JS.
- No npm dependencies. Anything more than what fits in one file is out of scope for this edition.
- No employee scheduling, payroll, customer-facing display, kitchen display, or online ordering.

---

## 2. Tech Choices

| Concern | Choice | Why |
|---|---|---|
| Container | One `.html` file | Zero install, double-click, runs anywhere. |
| Language | Plain ES2020+ JS, no transpilation | Modern browsers run it as-is. No build step. |
| Persistence | IndexedDB, accessed through a thin `db.*` wrapper | Real database, no size limit issues, abstracted for future swap. |
| Layout | Hand-written CSS with CSS variables for theming | No framework. Keeps the file self-contained and inspectable. |
| Fonts | Google Fonts (Fraunces, Inter, JetBrains Mono, Cairo for Arabic) | Loaded over the network once; cached by the browser. Falls back to system fonts if offline. |
| Charts | Hand-rolled SVG/CSS bars | Avoid pulling in chart libraries. |
| Money | Plain `number` for major units. All formatting through `fmt()` which respects the active currency's decimals. | |
| Time | ISO strings (`new Date().toISOString()`). Format only at display. | |
| i18n | Two flat dictionaries (`I18N.en`, `I18N.ar`) plus a `t(key)` helper. | Tiny, no library. |

---

## 3. File Anatomy

The single file is logically organized into seven concatenated sections so they can be extended one at a time. They live in the same file but are clearly delimited by comment banners.

```
brewbook.html
├── PART 1  — HTML shell + CSS (no JS)
│             <!doctype>, <head>, <style>, sidebar markup, main mount points,
│             toast and modal hosts.
│
├── PART 2  — Data Layer
│             • DB_NAME, DB_VERSION, STORES list
│             • db = { init, getAll, get, put, del, clear, raw }
│             • State (in-memory app state)
│             • CURRENCIES table
│             • UNIT_CONV table + convertQty()
│             • I18N dictionaries + t()
│             • toast(), modal(), closeModal()
│             • uid(), todayISO(), nowISO()
│             • fmt(), fmtNum()
│
├── PART 3  — Engine (accounting + inventory + order/purchase/expense lifecycle)
│             • ACCOUNTS chart of accounts
│             • postJournal, reverseJournalForRef
│             • getAccountBalance, getPnL, getBalanceSheet
│             • recordStockMove, consumeRecipeForOrder
│             • finalizeOrder (the special feature)
│             • finalizePurchase (weighted-avg cost)
│             • finalizeExpense
│             • nextOrderNum
│             • seedIfEmpty, reloadState
│
├── PART 4  — POS UI
│             renderPOS, product grid, cart, discount/payment/receipt modals
│
├── PART 5  — Inventory UI
│             renderProducts, renderRecipes, renderInventory + their editors
│
├── PART 6  — Finance UI
│             renderPurchases, renderExpenses, renderOrders
│
└── PART 7  — Reports + Settings + Bootstrap
              renderDashboard, renderPnL, renderBalanceSheet, renderSettings
              boot() — entry point
              </body></html>
```

Each part lives between banner comments:

```js
/* ============================================================
   PART 4  —  POINT OF SALE
   ============================================================ */
```

Use these banners to navigate. When extending, keep additions inside the right part. Don't merge parts.

**Build process for development:** during work, the file is split into seven sub-files (`part1_head.html` … `part7_dashboard.html`) and concatenated with `cat`. Final ship file is the concatenation. This is a developer-side convenience only; the deliverable is always one HTML file.

```bash
cat part1_head.html part2_data.html part3_engine.html \
    part4_pos.html part5_inventory.html part6_finance.html \
    part7_dashboard.html > brewbook.html
```

---

## 4. Data Layer

### 4.1 Stores (IndexedDB object stores)

All stores have `keyPath: 'id'`. Records are plain JSON. No indexes — scan-and-filter in JS, since data volume per shop stays small.

| Store | Record shape (key fields) |
|---|---|
| `settings` | `{ id:'app', currency, vatPct, shopName, address, phone, taxId, receiptFooter, language, openingCash, initialCapital }` |
| `categories` | `{ id, name, sortOrder }` |
| `products` | `{ id, sku, name, categoryId, price, taxable, active, recipe:[{materialId, qty, unit}] }` |
| `materials` | `{ id, sku, name, unit, costPerUnit, stock, reorderPoint, supplier }` |
| `suppliers` | `{ id, name, phone, notes }` |
| `purchases` | `{ id, date, supplierId, items:[{materialId, qty, unitCost}], total, paid, status }` |
| `expenses` | `{ id, date, category, vendor, amount, notes, paidFromAccount }` |
| `orders` | `{ id, num, date, mode, items:[…], subtotal, discount, vat, total, cogs, payment:{method, amountPaid, change}, cashier, status }` |
| `shifts` | `{ id, openedAt, closedAt, openedBy, openingFloat, closingCash, expectedCash, variance }` |
| `stockMoves` | `{ id, date, materialId, qty, type, refType, refId, costPerUnit, signedDelta }` |
| `journal` | `{ id, date, refType, refId, memo, reversed, lines:[{account, dr, cr, memo}] }` |
| `accounts` | (held in code as `ACCOUNTS` — store reserved for future custom accounts) |
| `users` | `{ id, name, role, pin }` |
| `shops` | `{ id, name, vat, address, phone, currency }` (reserved for multi-shop future) |

Notes:
- `recipe` is an embedded array on the product, not a separate store. This keeps the single-file edition simple. The migration to SQLite splits it into a `recipe_items` table.
- `journal.lines` is embedded for the same reason. Migration splits it into `journal_lines`.
- Soft delete is **not used** in v1 of the single-file edition. Hard delete only. (Migration to SQLite will add soft delete.)

### 4.2 The `db` Wrapper

Every read and write goes through this object. The application code never touches `indexedDB` directly. This is the seam for the future server swap.

```js
const db = (() => {
  let _db = null;
  return {
    init: () => Promise<IDBDatabase>,
    getAll: (store)            => Promise<Record[]>,
    get:    (store, id)        => Promise<Record | undefined>,
    put:    (store, obj)       => Promise<Record>,     // upsert
    del:    (store, id)        => Promise<void>,
    clear:  (store)            => Promise<void>,
    raw:    ()                 => IDBDatabase,
  };
})();
```

`init()` opens the database, creating any missing object stores on `onupgradeneeded`. Every subsequent call uses a fresh transaction.

### 4.3 In-Memory State

A single `State` object holds what the UI needs without re-reading IndexedDB on every render:

```js
const State = {
  page: 'pos',
  settings: null,
  categories: [],
  products: [],
  materials: [],
  cart: [],
  orderMode: 'dine-in',
  cashier: null,
  shop: null,
  selectedCat: null,
  cartDiscount: { type: 'percent', value: 0 },
  i18n: 'en',
};
```

`reloadState()` repopulates everything from the database. Call it after any write that changed records the UI lists. **Do not** mutate `State.products` etc. directly; always go through `db.put()` and then `reloadState()`.

### 4.4 Helpers (must exist, used everywhere)

- `uid()` — `id_${base36 timestamp}${random}`. CUID-ish, no library.
- `todayISO()` — `YYYY-MM-DD` for the local day.
- `nowISO()` — full ISO timestamp.
- `fmt(n)` — currency-formatted with the active currency's symbol and decimals.
- `fmtNum(n, decimals=2)` — number only.
- `convertQty(qty, fromUnit, toUnit)` — returns a number or `null` for incompatible units.
- `toast(msg, type)` — bottom-right notification, auto-dismiss 2.4s.
- `modal(html, opts)` — returns a Promise that resolves when `closeModal(value)` is called.

---

## 5. Currencies (12 supported)

`CURRENCIES` table in part 2:

| Code | Symbol | Name | Decimals |
|---|---|---|---|
| EGP | ج.م | Egyptian Pound (default) | 2 |
| SAR | ر.س | Saudi Riyal | 2 |
| AED | د.إ | UAE Dirham | 2 |
| USD | $ | US Dollar | 2 |
| EUR | € | Euro | 2 |
| GBP | £ | British Pound | 2 |
| KWD | د.ك | Kuwaiti Dinar | 3 |
| QAR | ر.ق | Qatari Riyal | 2 |
| BHD | د.ب | Bahraini Dinar | 3 |
| OMR | ر.ع | Omani Rial | 3 |
| JOD | د.أ | Jordanian Dinar | 3 |
| TRY | ₺ | Turkish Lira | 2 |

`fmt(n)` reads `State.settings.currency`, looks up the row, applies decimals and the symbol. Number positioned before the symbol with a space.

---

## 6. Unit Conversion

`UNIT_CONV` table:

```js
{
  g:    { base: 'g',  factor: 1 },
  kg:   { base: 'g',  factor: 1000 },
  ml:   { base: 'ml', factor: 1 },
  l:    { base: 'ml', factor: 1000 },
  pc:   { base: 'pc', factor: 1 },
  unit: { base: 'pc', factor: 1 },   // alias of pc
}

convertQty(qty, from, to) → qty in `to` units, or null if bases differ.
```

A recipe can specify ingredients in any compatible unit. The material is stored in its canonical unit. Conversion happens at consumption time.

---

## 7. Accounting Engine

### 7.1 Chart of Accounts (compile-time `ACCOUNTS` array)

Account ids are strings, stable forever, written into journal entries. Don't renumber.

```
1010  Cash on Hand                  asset
1020  Bank Account                  asset
1030  Accounts Receivable           asset
1100  Inventory — Raw Materials     asset
1500  Equipment & Fixtures          asset
1510  Accumulated Depreciation      asset (contra)
2010  Accounts Payable              liability
2020  VAT Payable                   liability
2030  Accrued Expenses              liability
2100  Loans Payable                 liability
3010  Owner's Capital               equity
3020  Retained Earnings             equity
4010  Sales Revenue                 revenue
4020  Discounts Given               revenue (contra)
5010  Cost of Goods Sold            expense
5100  Wastage & Spoilage            expense
6010  Rent                          expense
6020  Salaries & Wages              expense
6030  Utilities                     expense
6040  Marketing                     expense
6050  Supplies                      expense
6060  Maintenance                   expense
6070  Depreciation                  expense
6080  Other Expenses                expense
```

### 7.2 `postJournal(entry)`

```js
postJournal({
  date?: string,            // defaults to nowISO()
  refType: string,          // 'order' | 'order_cogs' | 'purchase' | 'purchase_pay'
                            // | 'expense' | 'opening' | 'opening_mat' | 'adjustment' | ...
  refId: string,
  memo?: string,
  lines: Array<{ account, dr?, cr?, memo? }>,   // dr and cr both ≥ 0; one is 0
}) → Promise<JournalRecord>
```

**Invariants enforced before write** (throw on violation):
1. `lines.length >= 2`.
2. `Math.abs(sum(dr) - sum(cr)) < 0.01` after rounding to 2 decimals.
3. Every account string exists in `ACCOUNTS`.

The function writes one record to `journal` containing the embedded `lines` array.

### 7.3 `reverseJournalForRef(refType, refId)`

Find every non-reversed entry matching `refType` and `refId`. For each, write a new entry tagged `refType: 'reverse_<original>'` with `dr` and `cr` swapped on each line; then mark the original `reversed = true`. Used for voids.

### 7.4 `getAccountBalance(accountId, asOfDate?)`

Sum dr and cr from every journal line in every non-reversed journal entry up to `asOfDate`. For debit-normal accounts (asset, expense) return `dr - cr`; for credit-normal (liability, equity, revenue) return `cr - dr`. Flip the sign if the account is a contra account.

### 7.5 `getPnL(fromDate?, toDate?)`

Returns:
```
{
  revenue:        { '4010': n, '4020': n },
  expense:        { '5010': n, '5100': n, '6010': n, ... },
  totalRevenue:   number,
  totalCogs:      number,    // 5010 + 5100
  totalOpex:      number,    // all other expense accounts
  grossProfit:    totalRevenue - totalCogs,
  netProfit:      grossProfit - totalOpex,
}
```

For revenue: contra accounts subtract (Discounts Given reduces Sales Revenue). For expense: straight `dr - cr` per account.

### 7.6 `getBalanceSheet(asOfDate?)`

Walks every non-revenue, non-expense account, gets its balance, and slots it into `asset`, `liability`, or `equity`. Suppresses zero balances except for `1010` and `3020` which always show.

Computes Retained Earnings live: `getPnL(null, asOfDate).netProfit`, added to (or set as) account `3020`.

Returns:
```
{
  asset:      [{ id, code, name, type, balance }],
  liability:  [...],
  equity:     [...],
  totalAssets, totalLiabilities, totalEquity: number,
}
```

The UI shows a green "✓ Books balanced" or red warning based on `|totalAssets − (totalLiabilities + totalEquity)| < 0.01`.

---

## 8. Inventory Engine

### 8.1 `recordStockMove(materialId, qty, type, refType, refId, costPerUnit?)`

`qty` is always positive in the call. Sign is derived from `type`:
- `in` or `adjust` (if qty is being added) → `+qty`
- `out`, `waste`, `adjust` (down) → `-qty`

Updates `materials.stock` and writes a `stockMoves` record. If `costPerUnit` is omitted, falls back to the material's current cost.

### 8.2 `consumeRecipeForOrder(order)`

For each line item in the order:
1. Load the product. Skip if it has no recipe.
2. For each ingredient: convert `ingredient.qty` from `ingredient.unit` to the material's `unit`. If incompatible, skip (and log).
3. Multiply by line `qty`. Call `recordStockMove(materialId, totalConsumed, 'out', 'order', order.id, m.costPerUnit)`.
4. Accumulate `totalConsumed * m.costPerUnit` into the order's COGS.

Returns the total COGS for the order.

### 8.3 Weighted-Average Cost on Purchase

Inside `finalizePurchase`, for each line:
```
oldVal      = stock * costPerUnit
newVal      = qty * unitCost
newStock    = stock + qty
m.costPerUnit = newStock > 0 ? (oldVal + newVal) / newStock : unitCost
m.stock       = newStock
```

### 8.4 `canMake(product, qty=1)`

Returns `true` if every ingredient has enough stock to produce `qty` units of the product. Used by the POS to grey out unavailable products and to block adding more to the cart.

---

## 9. The Special Feature: Order Lifecycle

This is the heart of the app. `finalizeOrder(orderDraft, payment)` does everything in sequence:

1. Build the order record: `id`, `num` (next order number), `date = nowISO()`, `payment`, `status = 'paid'`, `cashier = State.cashier?.id`.
2. Save the order to the `orders` store.
3. Call `consumeRecipeForOrder(order)` → returns `cogs`. Update the order with `cogs`, save again.
4. Post the **sales journal**:
   ```
   Dr  1010 Cash (or 1020 Bank, depending on payment.method)   total
       Cr 4010 Sales Revenue                                       subtotal
   Dr  4020 Discounts Given                                    discount   (only if > 0)
       Cr 2020 VAT Payable                                         vat    (only if > 0)
   ```
5. Post the **COGS journal** (only if cogs > 0):
   ```
   Dr  5010 Cost of Goods Sold     cogs
       Cr 1100 Inventory                cogs
   ```

These two journals are separate so the COGS line is auditable on its own.

`finalizePurchase` posts:
```
Dr  1100 Inventory                       total
    Cr 1010 Cash (if paid)               total
    OR
    Cr 2010 Accounts Payable (if not)    total
```
Plus stock movements in the same call.

`finalizeExpense` posts:
```
Dr  [chosen expense account]             amount
    Cr [paidFromAccount: 1010 / 1020 / 2010]    amount
```

---

## 10. Migration Path to Multi-Terminal (Node + SQLite)

The data layer is the only piece that needs to change. The migration is well-defined.

### 10.1 What stays the same
- Every UI rendering function in parts 4–7.
- Every engine function in part 3 (`postJournal`, `finalizeOrder`, `getPnL`, etc.) — they only call `db.*`.
- Every record shape in §4.1.
- Account ids, currency codes, unit codes, all of `lib`-equivalent constants.

### 10.2 What changes

Replace the `db` wrapper in part 2 with a `fetch()`-based client:

```js
const API = 'http://localhost:3001';
const db = {
  init:   () => fetch(`${API}/init`).then(r => r.json()),
  getAll: (store)         => fetch(`${API}/${store}`).then(r => r.json()),
  get:    (store, id)     => fetch(`${API}/${store}/${id}`).then(r => r.json()),
  put:    (store, obj)    => fetch(`${API}/${store}`, { method:'PUT',  headers:{'Content-Type':'application/json'}, body: JSON.stringify(obj) }).then(r => r.json()),
  del:    (store, id)     => fetch(`${API}/${store}/${id}`, { method:'DELETE' }),
  clear:  (store)         => fetch(`${API}/${store}`, { method:'DELETE' }),
};
```

Build a tiny Express server that exposes those endpoints against a SQLite database. Every store maps to a table; the embedded arrays (recipes on products, lines on journal entries) get split into junction tables. The mapping is mechanical:

| Store | SQLite tables |
|---|---|
| `products` | `products` + `recipe_items` |
| `journal` | `journal` + `journal_lines` |
| `purchases` | `purchases` + `purchase_items` |
| `orders` | `orders` + `order_items` |
| everything else | one table each |

The server reassembles embedded shapes on `getAll` / `get` so the client receives the same JSON it used to get from IndexedDB. **No engine code changes.**

The split tables and SQL schema for this server are fully specified in the companion `SPEC.md` for the Next.js edition. They match field-for-field.

### 10.3 Multi-terminal implications

Once on a server, multiple browsers can share one database. Two follow-up tasks become required (and only then):
- A locking discipline on `nextOrderNum()` — currently racy across terminals. Move it into the server inside the same transaction as the order insert.
- A cashier login screen — currently the cashier is just the seeded user.

Both are server-side concerns and don't touch the UI.

---

## 11. UI — Pages and Behaviour

### 11.1 Sidebar Navigation

Always visible (except on POS, which uses the full screen). Sections:

```
OPERATIONS
  ☕  Point of Sale     /pos
  📋  Orders            /orders
CONFIG
  🥐  Products
  📐  Recipes
  📦  Inventory
  🛒  Purchases
  💸  Expenses
ACCOUNTING
  📊  Dashboard
  📈  Profit & Loss
  ⚖️  Balance Sheet
  ⚙️  Settings
```

The sidebar shows shop name and current cashier at the bottom.

### 11.2 POS

- Two columns: product browser left (category tabs + product grid), cart panel right.
- Cart panel header: `Order #<next>` and live clock.
- Order mode pills: Dine-in / Takeaway / Delivery.
- Empty cart: "Tap a product to start an order".
- Product tile: name + price. Out-of-stock tiles are dimmed and disabled with a red "Out of stock" badge.
- Cart row: name, line total, qty `−` / number / `+`, Remove.
- Totals: subtotal, discount (if any, with % shown if percent), VAT (with % shown), grand total in display font.
- Buttons: `% Discount`, `Clear`, `Charge <total>`.
- Discount modal: type (percent / amount) + value.
- Payment modal: method (cash / card / wallet), amount tendered (auto-fills total for non-cash), live change display, Confirm button (disabled until tendered ≥ total for cash).
- On confirm: `finalizeOrder()`, show receipt modal, reset cart, refresh.

### 11.3 Receipt

Rendered inline in a modal sized like an 80mm thermal slip. Print button calls `window.print()`. Print stylesheet hides everything except the receipt.

Fields: shop name, address, phone, tax ID, order number, datetime, mode, cashier, line items with qty and line total, subtotal, discount, VAT, total, payment method + tendered + change, footer line.

### 11.4 Dashboard

KPIs (today sales, 7-day sales, MTD net profit with margin %, cash on hand, bank balance, inventory value, low-stock count). Two cards below: hourly bar chart for today's sales, top 8 products by revenue this month. Yellow alert card listing low-stock materials if any.

### 11.5 Orders

Date range filter (default today). Stats row: order count, sales (incl. VAT), VAT, COGS, average ticket. Table: #, datetime, mode, items, payment method, subtotal, VAT, total, COGS, margin %, View. View opens the receipt modal.

### 11.6 Products

Table: SKU, name, category, price, recipe cost (live), margin %, status, Edit, Recipe. Editor modal: SKU, name, category, price, taxable, active. Recipe action opens the Recipe editor.

### 11.7 Recipes

Table: product, ingredient count, recipe cost, sell price, food cost % (with green/amber/red threshold at 35% / 55%). Recipe editor (large modal): list of `{material, qty, unit}` rows. The unit dropdown is filtered to compatible units (mass / volume / count) based on the chosen material's base unit. Live recipe cost and food cost % update as the user edits. Save persists to `product.recipe`.

### 11.8 Inventory

Stat tiles: total SKUs, stock value (= sum of `stock * costPerUnit`), reorder alerts, out of stock. Table: SKU, name, supplier, stock, unit, cost/unit, stock value, status (OK/Reorder/Out), Edit. Buttons: New Material, Stock Adjustment.

Material editor: SKU, name, unit, cost per unit, reorder point, supplier. New materials have an "Opening Stock" field that posts an opening journal entry (`Dr 1100 / Cr 3010`).

Stock Adjustment modal: pick material, type (in / out / waste), qty, notes. Posts a journal entry:
- waste → `Dr 5100 / Cr 1100`
- out (correction down) → `Dr 6080 / Cr 1100`
- in (correction up) → `Dr 1100 / Cr 3010`

### 11.9 Purchases

Table: date, supplier, item count, total, status (paid / unpaid), Mark Paid (when unpaid). New Purchase modal: date, supplier, items rows (material + qty + unit cost), live total, "Paid in cash now" checkbox. Save runs `finalizePurchase` (stock update with weighted-avg cost, journal entry).

Mark Paid: posts `Dr 2010 / Cr 1010`, sets status.

### 11.10 Expenses

Table: date, category (account name), vendor, notes, paid from, amount. New Expense modal: date, amount, category (dropdown of expense accounts excluding `5010` and `5100`), paid from (Cash / Bank / Account Payable), vendor, notes. Save posts the journal.

### 11.11 P&L

Range presets: Today, Last 7 days, This month, YTD, All time, Custom. Format as a formal statement: Revenue (with discounts as contra), COGS (5010 + 5100), Gross Profit (with margin %), Operating Expenses, Net Profit (with margin %, green if positive / red if negative).

### 11.12 Balance Sheet

"As of" date picker. Three sections (Assets, Liabilities, Equity) listing each non-zero account. Total Liabilities + Equity row. Green confirmation or red warning if A ≠ L+E.

### 11.13 Settings

Two columns. Left: shop info form (name, address, phone, tax ID, currency, VAT %, language, receipt footer). Right top: Data Management — Export All Data (downloads a JSON of every store), Import Backup (file picker, confirms before replacing), Reset All Data (drops everything, re-seeds, after two confirmations). Right bottom: About box describing the feature set.

---

## 12. Design System

CSS variables at `:root`:

```
--bg: #faf7f2          (paper background)
--bg-2: #f3ede3        (subtle alt)
--ink: #1a1410         (primary text)
--ink-2: #4a3f35       (secondary)
--ink-3: #8a7e72       (muted)
--line: #e6ddcf
--paper: #fffdf8
--accent: #c2410c      (burnt orange — primary action)
--accent-2: #7c2d12    (deep coffee — hover/dark)
--accent-3: #166534    (money green — positive deltas)
--warn: #b45309
--bad: #b91c1c
--good: #166534
```

Fonts: `Fraunces` (display, weights 500–700, opsz 144, used in titles, large numbers, money totals), `Inter` (body), `JetBrains Mono` (numeric tables, receipts), `Cairo` (Arabic body when `body[dir="rtl"]`).

Components hand-rolled with classnames: `.btn`, `.btn-primary`, `.btn-accent`, `.btn-danger`, `.input`, `.field`, `.card`, `.tag`, `.tag.green/red/amber/blue`, `.stat`, `.tbl`, `.modal-overlay`, `.modal`, `.toast`. POS-specific: `.pos-layout`, `.cat-tab`, `.product-tile`, `.cart-panel`, `.cart-item`, `.qty-btn`. Finance-specific: `.fin-section`, `.fin-row.subtotal/total/indent`. Receipt: `.receipt`.

RTL: when `State.i18n === 'ar'`, set `document.body.dir = 'rtl'`. CSS uses logical properties (`padding-inline-start`, `margin-inline-end`, `text-align: start/end`) so layouts flip cleanly.

---

## 13. Seed Data

`seedIfEmpty()` runs at boot if no `settings/app` record exists. It creates:

- `settings/app` with `currency:'EGP'`, `vatPct:14`, `shopName:'Brewbook Café'`, sample address/phone, opening cash 1000, initial capital 50000, language `'en'`.
- 1 owner user (PIN 0000) and 1 cashier user (PIN 1234).
- 6 categories: Espresso, Brewed Coffee, Cold Drinks, Tea & Other, Pastries, Sandwiches.
- 20 raw materials with realistic Egyptian-market suppliers (Juhayna, Almarai, Cadbury, Domty, Daily Bakery, Lipton, Monin, Cairo Coffee Roasters, etc.).
- 21 products with full recipes wired to those materials.
- An opening journal entry: `Dr 1010 50,000 / Cr 3010 50,000`.
- An opening-inventory journal entry: `Dr 1100 <sum of stock × cost> / Cr 3010 <same>`.

After seed, the balance sheet must balance to the cent.

(For the exact list, see `seedIfEmpty()` in the reference `brewbook.html`. Any new build should replicate the same SKUs and prices to stay consistent with the prototype.)

---

## 14. Bootstrap

```js
async function boot() {
  try {
    await db.init();
    await seedIfEmpty();
    await reloadState();
    renderSidebar();
    await navigate('pos');
  } catch (err) {
    console.error('Boot failed:', err);
    document.getElementById('main').innerHTML =
      `<div style="padding:40px;color:#b91c1c">
         Failed to start: ${err.message}<br><br>
         If this persists, your browser may be blocking IndexedDB
         (e.g. private/incognito mode). Try a regular browser window.
       </div>`;
  }
}
window.addEventListener('DOMContentLoaded', boot);
```

The two failure modes worth handling explicitly: IndexedDB unavailable (incognito on some browsers, locked-down enterprise machines), and quota exceeded (very rare; tell the user to export and reset).

---

## 15. Test Plan (Manual + Headless)

The single-file edition has no unit-test runner. Verification is two paths:

### 15.1 Manual smoke test (5 minutes, every release)

1. Open `brewbook.html` in a fresh profile.
2. Confirm sample data loaded: 21 products visible on POS, dashboard shows starting balance.
3. Place an order: 2 cappuccinos, charge cash, exact tender. Confirm receipt appears. Print preview should look correct.
4. Open Inventory. Beans should be 4968 g (was 5000), milk should be 29700 ml (was 30000), cup12oz should be 998 (was 1000), lid 1998, sleeve 1498.
5. Open P&L (Today). Revenue 110.00, COGS 26.40, Gross Profit 83.60.
6. Open Balance Sheet. Green "✓ Books balanced" banner.
7. Settings → Export All Data. JSON file should download.
8. Settings → Import Backup with that JSON. App reloads with the same numbers.
9. Settings → currency to USD, VAT to 10. Sidebar and POS reflect the symbol; new orders use 10%.
10. Switch language to العربية. UI flips to RTL, labels translate.

### 15.2 Headless behavioural test (Node)

A `test.js` lives next to the file during development, not in the deliverable. It mocks `indexedDB`, evaluates the engine code via `vm.runInThisContext`, and runs the same scenarios as the manual test, asserting:
- After seed, balance sheet balances.
- After 1 order of 2 cappuccinos: stock decremented by exactly 32g beans + 300ml milk + 2 cups12oz + 2 lids + 2 sleeves; COGS 26.40; books still balanced.
- After a purchase: weighted-average cost matches `(oldStock × oldCost + qty × unitCost) / (oldStock + qty)`.
- After a 5,000 EGP rent expense: net profit drops by 5,000; cash drops by 5,000; books still balanced.

Mock implementation note: real IndexedDB returns deserialized copies on each read; an in-memory `Map` returns the same reference. When asserting "stock before vs after", clone the value (`structuredClone`) — don't hold a reference.

---

## 16. Conventions and Rules for Claude Code

- **Never** introduce a build step or a dependency that requires npm install. The deliverable is one HTML file that opens by double-click.
- **Never** access `indexedDB` directly outside the `db` wrapper.
- **Never** mutate State arrays in place after reading from them; always `db.put()` then `reloadState()`.
- **Always** post a journal entry for any change that affects an account balance — sale, purchase, expense, stock adjustment, opening balance, void.
- **Always** validate `sum(dr) == sum(cr)` before writing a journal record. Throw if not.
- **Always** route money values through `fmt()` for display. Never concatenate the symbol manually.
- **Always** use `convertQty()` when the recipe unit and material unit may differ.
- **Always** keep the seven-part structure when adding code. Don't merge banners.
- **Never** hard-code English in the UI. Use `t('key')` and add the key to both `I18N.en` and `I18N.ar`.
- When in doubt about behavior, open the reference `brewbook.html` and match it.

---

## 17. Future Work (when the time comes)

- Move to the Node + SQLite server (see §10) once a second terminal is needed.
- Add cashier login + PIN screen.
- Add shift open/close with cash drawer reconciliation (the `shifts` store is already reserved).
- Add customer accounts and loyalty points.
- Add a kitchen display by mirroring `orders` to a second view filtered by `status: 'open'`.
- Add multi-shop support by adding a `shopId` column to every table and a shop switcher.

None of these are needed for v1. Don't build for them.

---

## 18. Reference

`brewbook.html` is the working reference build that ships alongside this spec. It is the source of truth for visual design, copy, and behavioral details that this document does not cover explicitly. Everything in this spec was extracted from that file, and a faithful re-implementation should produce identical observable behavior.

End of spec.
