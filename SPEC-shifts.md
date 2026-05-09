# Brewbook — Shift Management Addendum

This addendum defines a complete shift-management feature: PIN login, opening cash float, per-shift sales attribution, manager-authorized voids, and a printed Z-report at close with cash variance and category/top-seller breakdowns.

It supplements both `SPEC.md` (Next.js + SQLite edition) and `SPEC-singlefile.md` (single-file HTML edition). The behavior is identical; the implementation differs only at the data layer. Where the two editions diverge, both versions are given.

---

## 1. Overview

A **shift** is the period one cashier operates the till between login and close-out. Every order placed during a shift is tagged with `shiftId` and `cashierId`. At close, the POS reconciles cash in the drawer against expected cash, prints a Z-report on the receipt printer, and locks the shift (no further edits to its orders).

Lifecycle:

```
Login (PIN) → Open Shift (enter float) → Run Orders → [Manager Voids] → Close Shift (count cash) → Print Z-Report → Locked
```

A user cannot place orders without an open shift. Two cashiers cannot have an open shift on the same terminal at the same time.

---

## 2. Roles and Permissions

| Role | Login | Open shift | Place orders | Void order | Close shift | Reopen closed shift | Reports |
|---|---|---|---|---|---|---|---|
| `cashier` | ✓ | ✓ | ✓ | — (must request manager) | ✓ (own only) | — | own shift only |
| `manager` | ✓ | ✓ | ✓ | ✓ (any open shift) | ✓ (any) | ✓ (within 24h) | all |
| `owner` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (anytime) | all |

The user `role` field already exists in the `users` table; no schema change for roles.

**Authentication.** Each user has a 4–6 digit PIN, hashed with bcrypt (Next.js edition) or SHA-256 with per-user salt (single-file edition — the salt is the user's `id`, which is unique). PINs never appear in plaintext anywhere except the entry input.

---

## 3. Data Model Changes

### 3.1 New / changed tables

**`shifts`** — already reserved in both SPECs. Promoted to a real working table.

```ts
shifts {
  id                  text pk
  num                 integer unique     // sequential, like order numbers, per terminal
  cashier_id          text fk → users.id
  opened_at           text not null      // ISO timestamp
  closed_at           text               // null while open
  opening_float       real not null      // cash placed in drawer at open
  expected_cash       real               // computed at close = float + cash_sales − cash_refunds
  counted_cash        real               // entered by cashier at close
  variance            real               // counted − expected (positive = over, negative = short)
  status              text not null      // 'open' | 'closed' | 'reopened'
  closed_by           text fk → users.id // who pressed close (may differ from cashier)
  notes               text
  z_report_json       text               // snapshot of the Z-report at close (JSON), for reprint
  reopened_at         text
  reopened_by         text fk → users.id
  reopen_reason       text
  created_at, updated_at
}
```

**`orders`** — add three columns:

```
shift_id            text fk → shifts.id      not null
voided_at           text                                      -- already in SPEC.md, keep
voided_by           text fk → users.id                        -- already in SPEC.md, keep
voided_reason       text                                      -- already in SPEC.md, keep
manager_void_id     text fk → users.id                        -- who authorized the void (may differ from voided_by)
```

`voided_by` is the user who pressed Void. `manager_void_id` is the manager who entered their PIN to authorize. Often the same person. Always recorded separately.

**`users`** — adjust:

```
pin_hash            text not null     -- replaces the prototype's plaintext `pin`
pin_salt            text not null     -- per-user random salt
last_login_at       text
failed_attempts     integer not null default 0
locked_until        text              -- if too many failed attempts, lock for N minutes
```

Migration note: the seed creates owner with PIN `0000` and cashier with PIN `1234` — keep those defaults so manual smoke tests still work, but hash them.

### 3.2 Single-file edition

The single-file edition stores `shifts` and the new fields directly on the existing IndexedDB stores. No schema migration is required (IndexedDB is schemaless), but `seedIfEmpty()` must be updated to:
- Hash the seeded PINs.
- Create no shift (the first cashier opens one on first use).

---

## 4. Engine — New Service Functions

All of these go in part 3 (single-file edition) or `server/services/shifts.ts` (Next.js edition).

### 4.1 `hashPin(pin, salt)` and `verifyPin(pin, user)`

```
hashPin(pin: string, salt: string): string
verifyPin(pin: string, user: User): boolean
```

- Single-file: `SubtleCrypto.digest('SHA-256', salt + pin)` → hex.
- Next.js: `bcrypt.hash` and `bcrypt.compare`. Wrap in async functions.

The login flow rejects after 5 failed attempts in a row, locks the account for 5 minutes (`locked_until = now + 5min`). Successful login resets `failed_attempts` to 0.

### 4.2 `login(pin)`

Iterates active users, finds the first whose hashed PIN matches. Throws on no match or on a locked account. Returns the user. Sets `State.cashier = user` (single-file) or sets a session cookie (Next.js).

Why iterate? Cashiers don't enter a username — just a PIN. PIN collisions across users must be prevented at user-create time: if two active users would share a PIN, reject the second one.

### 4.3 `openShift(cashierId, openingFloat)`

```
openShift(cashierId: string, openingFloat: number): Promise<Shift>
```

1. Reject if any open shift already exists for this terminal (single-file: any open shift in DB; Next.js: any open shift on this terminal — see §4.7).
2. Generate `num = max(shifts.num) + 1`.
3. Insert the shift with `status: 'open'`, `opened_at: now`, `opening_float`.
4. Post journal entry — none required at open. The cash float is already in `1010 Cash on Hand`; we're just attributing it to a shift, not moving it. *(If the float is actually being added to the till from a separate safe, that's a transfer the manager records as a separate cash move; not part of openShift.)*
5. Audit-log `shift_open`.

Returns the new shift. UI sets `State.activeShift = shift`.

### 4.4 `voidOrder(orderId, reason, managerPin)`

```
voidOrder(orderId: string, reason: string, managerPin: string): Promise<void>
```

1. Verify `managerPin` against an active user with role `manager` or `owner`. Throw on mismatch.
2. Load the order. Reject if `status !== 'paid'` or if its shift is `closed`.
3. **Reverse the journals.** Call `reverseJournalForRef('order', orderId)` and `reverseJournalForRef('order_cogs', orderId)`. This writes mirror entries that cancel the original sale and COGS postings.
4. **Restore the inventory.** For every `stockMoves` row matching `refType: 'order', refId: orderId`, write an opposite move (`type: 'in'`, same qty, ref `'order_void'`).
5. Update the order: `status: 'voided'`, `voided_at: now`, `voided_by: State.cashier.id`, `manager_void_id: <manager>.id`, `voided_reason: reason`.
6. Audit-log `order_void`.

The order stays in the database (never delete) — the Z-report needs to show voids.

### 4.5 `getShiftSummary(shiftId)`

Reads everything needed for the Z-report. Returns:

```ts
{
  shift: { id, num, openedAt, closedAt?, openingFloat, cashier: {id, name} },
  totals: {
    orderCount: number,
    voidedCount: number,
    grossSales: number,         // sum of (subtotal + vat) for paid orders
    netSales: number,           // gross − discounts (so revenue line on books)
    discounts: number,
    vat: number,
    cogs: number,
    refunds: number,            // total of voided orders' totals (cash refunded if cash, etc.)
  },
  byPayment: {
    cash:   { count, gross, refunds, net },     // net = gross − refunds; this is what cashier owes the drawer
    card:   { count, gross, refunds, net },
    wallet: { count, gross, refunds, net },
  },
  byCategory: Array<{ categoryId, categoryName, qty, revenue }>,  // sorted desc by revenue
  topSellers: Array<{ productId, name, qty, revenue }>,           // top 10 desc by revenue
  voids: Array<{                                                  // every voided order in this shift
    num, voidedAt, total, reason, voidedByName, managerName
  }>,
  cash: {
    openingFloat: number,
    expectedCash: openingFloat + byPayment.cash.net,
    countedCash?: number,    // null until close
    variance?: number,       // null until close
  }
}
```

This function is called twice per shift: once at close (computed and stored as `z_report_json`), and any time the user reprints (read directly from `z_report_json`).

### 4.6 `closeShift(shiftId, countedCash, notes, closingUserId)`

1. Reject if `status !== 'open'`.
2. Compute `summary = getShiftSummary(shiftId)`.
3. `expectedCash = openingFloat + summary.byPayment.cash.net`.
4. `variance = countedCash − expectedCash`.
5. **Post journal entry for variance**, only if `|variance| ≥ 0.01`:
   - If short (variance < 0): `Dr 6080 Other Expenses (variance abs) / Cr 1010 Cash`.
   - If over (variance > 0): `Dr 1010 Cash (variance) / Cr 6080 Other Expenses` (negative expense, effectively income).
   - Memo: `"Shift #N cash variance"`.
6. Update the shift: `closed_at, closed_by, expected_cash, counted_cash, variance, status: 'closed', notes, z_report_json: JSON.stringify({...summary, cash:{...summary.cash, countedCash, variance}})`.
7. Audit-log `shift_close`.
8. UI: clear `State.activeShift`, log out the cashier (force re-login for next shift), open the print dialog for the Z-report.

### 4.7 `reopenShift(shiftId, reason, managerPin)` (manager/owner only)

Edge case for fixing mistakes. Conditions:
- Shift was closed less than 24 hours ago.
- No newer shift has been closed after it (shifts close in order; can't reopen the second-to-last).
- Manager or owner authorizes with PIN.

Sets `status: 'reopened', reopened_at, reopened_by, reopen_reason`. The shift can now accept voids again. A subsequent close re-runs the variance calculation and posts a correcting journal entry if needed.

### 4.8 Multi-terminal note

The single-file edition runs on one machine, so "the open shift" is whichever shift in the store has `status: 'open'`. When migrated to the server, terminals share a database, so two cashiers can have shifts open simultaneously — one per terminal. Add a `terminal_id` column to `shifts` and scope `openShift`'s "no open shift exists" check to `terminal_id`.

---

## 5. UI — Screens and Flows

### 5.1 Lock Screen

When `State.activeShift` is null, the entire app is replaced by a full-screen lock view:

```
┌────────────────────────────────┐
│      ☕  Brewbook               │
│                                │
│      Enter Cashier PIN         │
│                                │
│      [ • • • • ]               │
│                                │
│      [1] [2] [3]               │
│      [4] [5] [6]               │
│      [7] [8] [9]               │
│      [⌫] [0] [✓]               │
│                                │
│      Locked since 14:32        │
└────────────────────────────────┘
```

Numeric keypad rendered on screen (works on touchscreens and tablets); also accepts physical keyboard input. PIN field shows dots, never digits. On submit, call `login(pin)`. On success, go to **Open Shift** screen if no open shift exists, or directly to POS if the cashier has a resumable open shift (e.g. browser was closed mid-shift). On failure, shake animation, show "Wrong PIN" without revealing whether a user with that PIN exists.

The lock screen is the first view on app boot. It also re-appears immediately after `closeShift`.

### 5.2 Open Shift Modal

Shown after login if no open shift exists for this cashier:

```
Welcome, Ahmed.

Cash float in drawer:  [____] EGP
Notes (optional):       [_________________]

[ Start Shift ]   [ Cancel ]
```

"Cancel" returns to the lock screen. "Start Shift" calls `openShift(cashierId, float)` and routes to POS.

### 5.3 POS — additions

A small bar above the cart shows the active shift:

```
Shift #42 • Ahmed • opened 14:32         [End Shift]
```

The "End Shift" button opens the **Close Shift** flow.

### 5.4 Manager Void Dialog

When a cashier presses "Void" on an order in the Orders page:

```
Void Order #128

Reason for void:
[____________________________________]

Manager authorization required.
Manager PIN: [ • • • • ]

[ Cancel ]   [ Void Order ]
```

On confirm, call `voidOrder(orderId, reason, managerPin)`. Show the variance result in a toast:
- "Order voided. EGP 110.00 will be refunded to cash drawer."

Voided orders show with a strikethrough and red `Voided` tag in the orders list. The Z-report lists them under a "Voids" section.

### 5.5 Close Shift Flow

Two-step modal.

**Step 1 — Cash Count**
```
Close Shift #42

Opening float:           200.00 EGP
Cash sales this shift: + 1,245.40 EGP
                         ─────────
Expected in drawer:      1,445.40 EGP

Count the cash drawer and enter the actual amount:
Counted cash:           [_______] EGP

Notes (optional):
[________________________________]

[ Back ]   [ Continue → ]
```

The "Expected" amount is shown only after the cashier types a counted value, OR shown immediately as a "reveal" the cashier toggles. In practice, shops often hide the expected amount during count to prevent fudging — make it a setting (`Settings → Hide expected cash during shift close`). Default: hide.

**Step 2 — Confirm and Print**
```
Variance:    Counted 1,440.00 − Expected 1,445.40 = -5.40 EGP (Short)

Are you sure you want to close this shift?
Once closed, only a manager can reopen it within 24 hours.

[ Back ]   [ Close & Print Z-Report ]
```

On confirm, call `closeShift(...)`. On success, immediately open the printable Z-report view and trigger `window.print()`. Cashier is logged out; lock screen returns.

### 5.6 Z-Report Layout

Renders to a print-friendly route (`/print/zreport/[shiftId]` in Next.js, or a `data-page="zreport"` route in single-file). Sized for 80mm thermal paper.

```
================================
       BREWBOOK CAFÉ
   12 El-Tahrir Street, Cairo
        +20 100 000 0000
================================
       Z-REPORT (CLOSE)

Shift #42
Cashier:    Ahmed  (ID 4F2A)
Opened:     2026-05-09  14:32
Closed:     2026-05-09  22:18
Closed by:  Manager Sara

────────────────────────────────
ORDERS

Total orders:                 47
Voided orders:                 2
Net order count:              45

────────────────────────────────
SALES

Gross sales (incl. VAT)  4,820.40
Discounts given            −85.00
VAT collected              591.92
Net revenue              4,143.48
COGS                     1,236.10
Gross profit             2,907.38

────────────────────────────────
BY PAYMENT METHOD

           Count    Gross   Refunds    Net
Cash         28   1,445.40    0.00  1,445.40
Card         15   2,375.00    0.00  2,375.00
Wallet        2   1,000.00    0.00  1,000.00
                              ────────────
Total         45                    4,820.40

────────────────────────────────
BY CATEGORY

  Espresso          22  qty   1,420.00
  Brewed Coffee      8  qty     320.00
  Cold Drinks       12  qty     780.00
  Pastries           9  qty     360.00
  Sandwiches         3  qty     180.00
  Tea & Other        4  qty     100.00

────────────────────────────────
TOP SELLERS

1.  Cappuccino        15×    825.00
2.  Latte             10×    600.00
3.  Iced Latte         8×    520.00
4.  Espresso (Single)  6×    180.00
5.  Plain Croissant    5×    175.00
6.  Hot Chocolate      4×    200.00
7.  Caramel Macchiato  3×    210.00
8.  Cheese Sandwich    2×    120.00

────────────────────────────────
VOIDS

#128  Cappuccino +1     -125.40
      Reason: customer changed mind
      Voided by Ahmed, auth Sara

#136  Latte             -68.40
      Reason: wrong drink
      Voided by Ahmed, auth Sara

────────────────────────────────
CASH RECONCILIATION

Opening float           +  200.00
Cash sales (net)        +1,445.40
                         ────────
Expected in drawer       1,645.40
Counted cash             1,640.00
                         ────────
Variance                    −5.40   SHORT

────────────────────────────────
        ═══ END OF Z-REPORT ═══
        Printed 22:18 on 2026-05-09

  Cashier signature: ____________
  Manager signature: ____________

================================
```

The report is a snapshot. It is computed at `closeShift` time and stored in `z_report_json`, so reprinting later produces the exact same report even if subsequent journal corrections occur.

### 5.7 Reprint and Reopen

Under `Reports → Shifts`, list every closed shift with: number, cashier, dates, totals, variance. Each row has:
- **Reprint Z-Report** — opens the same print view from the snapshot.
- **Reopen** (manager/owner only, within 24h) — opens the manager-PIN dialog and `reopenShift`.

### 5.8 Settings — additions

```
Shift Management
─────────────────
☐  Hide expected cash during shift close       (default: checked)
☐  Require manager PIN for every void          (default: checked)
   Float threshold:  [200.00] EGP — minimum opening float
   Lock account after [5] failed PIN attempts for [5] minutes
```

---

## 6. Print Stylesheet for Z-Report

```css
@media print {
  body * { visibility: hidden; }
  .z-report, .z-report * { visibility: visible; }
  .z-report {
    position: absolute; left: 0; top: 0;
    width: 80mm;
    font-family: 'JetBrains Mono', monospace;
    font-size: 11px;
    line-height: 1.4;
    padding: 4mm;
  }
  .z-report .row { display: flex; justify-content: space-between; }
  .z-report .sep { border-top: 1px dashed #000; margin: 4px 0; }
  @page { size: 80mm auto; margin: 0; }
}
```

---

## 7. Schema Changes Summary

### Single-file edition

In part 3 `STORES` array — `shifts` was already listed, no change needed. Update `seedIfEmpty()` to:
- Hash the seeded PINs (replace `pin: '0000'` with `pinSalt + pinHash` fields).
- Skip auto-assigning `State.cashier`. The boot flow now goes to the lock screen, not POS.

### Next.js edition

Add a Drizzle migration with:

```sql
ALTER TABLE shifts
  ADD COLUMN num integer,
  ADD COLUMN expected_cash real,
  ADD COLUMN counted_cash real,
  ADD COLUMN variance real,
  ADD COLUMN closed_by text REFERENCES users(id),
  ADD COLUMN z_report_json text,
  ADD COLUMN reopened_at text,
  ADD COLUMN reopened_by text REFERENCES users(id),
  ADD COLUMN reopen_reason text;

CREATE UNIQUE INDEX idx_shifts_num ON shifts(num);
CREATE INDEX idx_shifts_status ON shifts(status);

ALTER TABLE orders
  ADD COLUMN shift_id text REFERENCES shifts(id),
  ADD COLUMN manager_void_id text REFERENCES users(id);

CREATE INDEX idx_orders_shift ON orders(shift_id);

ALTER TABLE users
  RENAME COLUMN pin TO pin_legacy;
ALTER TABLE users
  ADD COLUMN pin_hash text,
  ADD COLUMN pin_salt text,
  ADD COLUMN last_login_at text,
  ADD COLUMN failed_attempts integer NOT NULL DEFAULT 0,
  ADD COLUMN locked_until text;

-- Migration step: hash existing pin_legacy values into pin_hash + pin_salt
-- Then drop pin_legacy in a second migration.
```

---

## 8. Tests

### Unit (Next.js edition, Vitest)

`tests/unit/shifts.test.ts`:

1. Open shift with float 200 → `status='open'`, `expected_cash=null`, `opening_float=200`.
2. Cannot open second shift while one is open → throws.
3. Place 2 orders (cash 100, card 150) in the shift → `byPayment.cash.gross=100`, `byPayment.card.gross=150`.
4. Void one cash order with manager PIN → reverses journals, restores stock, `voids` array populated.
5. Wrong manager PIN on void → throws, no DB change.
6. Close shift with counted=297 (after 100 cash sale and 200 float) → variance −3, journal posts `Dr 6080 3 / Cr 1010 3`, books still balance.
7. Close shift with counted=305 → variance +5, journal posts `Dr 1010 5 / Cr 6080 5`.
8. Reopen within 24h → status `reopened`, can void another order.
9. Reopen after 24h → throws.
10. Z-report snapshot matches live computation immediately after close.

### E2E (Playwright)

`tests/e2e/shift-flow.spec.ts`:

1. App boots to lock screen (no shift).
2. Enter PIN 1234 → cashier logged in.
3. Enter float 200, start shift.
4. Place 2 cappuccinos, pay cash exact.
5. Place 1 latte, pay card.
6. Open Orders tab, click Void on the latte → enter reason → enter manager PIN → confirm.
7. Click "End Shift" in POS bar.
8. Cash count screen: counted = 200 + (cap1 + cap2) cash total. Continue.
9. Variance is 0. Confirm. Print preview opens.
10. Z-report shows: 3 orders / 1 void / cash count matches expected / category and top-sellers populated.
11. Lock screen returns.

---

## 9. Build Order (slot into existing build sequence)

For the Next.js edition, insert after step 9 (admin pages built) in `SPEC.md` §15:

> **9.5 Shift management.** Add the schema migration. Implement `login`, `openShift`, `voidOrder`, `getShiftSummary`, `closeShift`, `reopenShift`. Build the lock screen, open-shift modal, manager-PIN dialog, close-shift flow, Z-report print view, and Reports → Shifts list. Wire the POS gating (no orders without active shift). Tests pass. Commit.

For the single-file edition, insert as a new part:

> Add a **PART 3.5 — Auth & Shifts** block to `brewbook.html`, located between part 3 (engine) and part 4 (POS). It contains the auth helpers, the shift functions, the lock screen rendering, and the close-shift flow. Update `boot()` to route to the lock screen when there's no `State.cashier`, and update POS rendering to show the shift bar and require an active shift.

---

## 10. Conventions for Claude Code

- Never accept a PIN over the wire as plaintext beyond the immediate verify call. Hash, compare, discard.
- Never delete a voided order. Voids stay in the database forever for audit.
- Never recompute the Z-report after close from live data. Always read the stored `z_report_json` snapshot. (Live data may have changed due to subsequent corrections; the printed report represents what was true at close.)
- The variance journal entry is **mandatory** if `|variance| ≥ 0.01` — otherwise the books drift out of balance.
- A cashier's session ends at `closeShift`. Force re-login. Do not allow same-session reopen.
- The shift bar in POS is ambient context; it must always show `Shift #N`, cashier name, and an "End Shift" button. Removing it breaks the cashier's mental model of where they are.

End of addendum.
