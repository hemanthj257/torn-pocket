# Budget tracker app — technical spec (v1)

Local-only Android app. No backend server, no network calls — everything lives in Room (SQLite) on the phone. "API" below means the internal Repository/DAO functions your UI calls, not a network API.

---

## 1. Data layer — tables (Room entities)

### `Wallet`

| Field | Type | Notes |
| --- | --- | --- |
| `id` | Long (PK, autogen) |  |
| `name` | String | e.g. "Food", "Transport" |
| `budgetLimit` | Double | monthly limit |
| `currentSpent` | Double | sum of CONFIRMED transactions only |
| `linkedAppPackage` | String | e.g. `com.google.android.apps.nbu.paisa.user` |
| `createdAt` | Long (epoch) |  |

### `Category`

| Field | Type | Notes |
| --- | --- | --- |
| `id` | Long (PK, autogen) |  |
| `name` | String | user-defined, e.g. "Groceries", "Dining out", "Fuel" |
| `createdAt` | Long (epoch) |  |

### `Transaction`

| Field | Type | Notes |
| --- | --- | --- |
| `id` | Long (PK, autogen) |  |
| `walletId` | Long (FK → Wallet) |  |
| `categoryId` | Long? (FK → Category, nullable) | finer-grained tag within the wallet |
| `reason` | String | free text |
| `amount` | Double | locked in at entry |
| `status` | Enum | `PENDING` / `CONFIRMED` / `CANCELLED` |
| `createdAt` | Long (epoch) |  |
| `confirmedAt` | Long? (epoch, nullable) |  |

Assumption: category tags individual transactions, not the wallet itself — so a Food wallet can have spends split across "Groceries" / "Dining out" / "Delivery" for finer reporting later. `categoryId` is nullable so you're never blocked from logging a transaction just because you haven't picked a category yet. Three tables now, two foreign keys.

---

## 2. Internal API — DAO + Repository functions

### `WalletDao`

| Function | Purpose |
| --- | --- |
| `insertWallet(wallet)` | create wallet |
| `getAllWallets(): Flow<List<Wallet>>` | reactive list for Home screen |
| `getWalletById(id): Flow<Wallet>` | for detail/edit |
| `updateSpent(id, newAmount)` | called only on confirm |
| `updateWallet(wallet)` | edit name/limit/linked app |

### `CategoryDao`

| Function | Purpose |
| --- | --- |
| `insertCategory(category)` | add a custom category |
| `getAllCategories(): Flow<List<Category>>` | populates the category dropdown |
| `deleteCategory(id)` | remove unused category |

### `TransactionDao`

| Function | Purpose |
| --- | --- |
| `insertTransaction(txn)` | creates in `PENDING` state |
| `updateStatus(id, status, confirmedAt)` | PENDING → CONFIRMED/CANCELLED |
| `getPendingForWallet(walletId): Flow<List<Transaction>>` | used in budget validation |
| `getAllPending(): Flow<List<Transaction>>` | for the "clear your pending" banner |
| `getTransactionsForWallet(walletId): Flow<List<Transaction>>` | history view |
| `deleteTransaction(id)` | only for cancelled/stale cleanup |

### `TransactionRepository` (the layer your ViewModels actually call)

| Function | Purpose |
| --- | --- |
| `getRemainingBudget(walletId): Double` | `budgetLimit − currentSpent − sum(PENDING for wallet)` |
| `createTransaction(reason, amount, walletId): Result` | validates against remaining budget → inserts `PENDING` → returns success/warning/blocked |
| `confirmTransaction(id)` | **@Transaction**: re-validates, sets `CONFIRMED`, increments `wallet.currentSpent` — atomic |
| `cancelTransaction(id)` | sets `CANCELLED`, no balance change |
| `launchPaymentApp(context, packageName): Boolean` | wraps `PackageManager.getLaunchIntentForPackage`; returns false if app not installed |

This Repository is your "backend" — it's the only thing that talks to the DAOs. ViewModels never call DAOs directly.

---

## 3. Frontend — Compose screens

| Screen | Shows | Key actions |
| --- | --- | --- |
| **HomeScreen** | Wallet cards (name, remaining/limit), pending-transactions banner if any exist | tap wallet → detail; tap "+" → NewTransaction; tap banner → PendingTransactions |
| **NewTransactionScreen** | Reason field, wallet dropdown, category dropdown (optional, with inline "+ new category"), amount field, live remaining-budget preview, validation message | "Open \[App\]" button → creates PENDING txn → fires intent |
| **ConfirmSheet** (bottom sheet, triggered from `onResume`) | "Paid ₹X via \[App\]?" | Confirm / Edit amount / Cancel |
| **WalletSetupScreen** | Name, budget limit, linked app picker (list of installed UPI apps) | Save / Delete wallet |
| **CategorySetupScreen** | List of custom categories | Add / rename / delete category |
| **PendingTransactionsScreen** | List of all PENDING txns across wallets, with age ("2 days ago") | Confirm / Cancel each |

---

## 4. ViewModels (one per screen, state via `StateFlow`)

| ViewModel | Owns |
| --- | --- |
| `HomeViewModel` | wallet list, pending count, remaining budgets |
| `NewTransactionViewModel` | form state, live validation result, selected wallet |
| `WalletSetupViewModel` | form state for create/edit |
| `PendingTransactionsViewModel` | list of pending txns, confirm/cancel actions |

Each ViewModel calls `TransactionRepository` / `WalletDao` — never touches Room directly.

---

## 5. Middleware — validation rules

Not a separate architectural layer; enforced inside `TransactionRepository` before any write:

1. **Amount > 0** — reject zero/negative at form level.
2. **Budget check on create**: `amount > getRemainingBudget(walletId)` → block or warn (your call: hard block vs. "this'll go negative, continue?" confirm).
3. **Budget re-check on confirm**: re-run the same check in case other pending txns confirmed in between.
4. **App-installed check**: before showing "Open \[App\]" button, verify `getLaunchIntentForPackage` doesn't return null — else show "app not installed, update wallet settings."

---

## 6. Navigation graph

```
Home ──► NewTransaction ──► (external: payment app) ──► onResume ──► ConfirmSheet ──► back to Home
  │
  ├──► WalletSetup (create/edit)
  └──► PendingTransactions
```

---

## 7. Backend / hosting

**None.** Fully local — Room/SQLite on-device, no server, no hosting cost, works offline.

Trade-off to be aware of: no cross-device sync, no automatic backup. If the phone is lost/wiped, the ledger is gone unless you add Room's export-to-file or Google auto-backup later — worth flagging now even though it's out of scope for v1.