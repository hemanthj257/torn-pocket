# Implementation Plan — Offline Budget Tracker (Phase 1)

> **Note on location:** Plan mode restricts writes to `/home/ubuntu/.opencode/plan`, so this file was
> written here instead of `docs/PLAN.md`. To put it where you asked:
> `mv /home/ubuntu/.opencode/plan/PLAN.md /workspace/docs/PLAN.md`

Derived from [`SPEC.md`](./SPEC.md). Plan only — no application code has been written yet.

Every version number below was verified against this machine and against live Google Maven /
Maven Central metadata on the day this plan was written (not from memory).

**Contents**

1. [Environment findings](#1-environment-findings)
2. [Project setup](#2-project-setup)
3. [Package & folder structure](#3-package--folder-structure)
4. [Build order](#4-build-order)
5. [The three correctness guarantees](#5-the-three-correctness-guarantees)
6. [Material 3 Expressive theming](#6-material-3-expressive-theming--what-actually-exists)
7. [`onResume` → ConfirmSheet reliability](#7-onresume--confirmsheet-reliability)
8. [Open questions & decisions needed](#8-open-questions--decisions-needed-before-code)
9. [On approval — what happens next](#9-on-approval--what-happens-next)

---

## 1. Environment findings

These constrain the plan, so they come first.

| Fact | Value | Impact |
|---|---|---|
| Android SDK | `/opt/android-sdk` — **only** `platforms/android-36` + `build-tools/36.0.0`, licenses accepted | `compileSdk` / `targetSdk = 36` works as requested. Any newer platform (e.g. 37) is **not installed** — must pin 36 explicitly. |
| JDK | 17.0.20 (only JDK present) | Hard ceiling on AGP choice. AGP 9.x wants a newer JDK; **AGP 8.13.2 is the safe pick** (see §2). |
| Gradle | not installed, no `~/.gradle`, no `~/.m2` | Project must ship a Gradle **wrapper**; the first build downloads the distribution and all dependencies. Network access to `repo1.maven.org` and `dl.google.com` is **confirmed working**. |
| Emulator | **no `emulator/`, no `system-images/`** | Instrumented `androidTest` cannot be executed on this machine. All automated testing must be JVM / Robolectric. See §8 Q12. |

---

## 2. Project setup

### 2.1 Toolchain

| Piece | Version | Why this one |
|---|---|---|
| Gradle wrapper | **8.13.x** (floor required by AGP 8.13.2) | Works on JDK 17. |
| Android Gradle Plugin | **8.13.2** | Latest 8.x stable. AGP's latest stable is 9.4.1, but the AGP 9 line raises the Gradle/JDK floor above the JDK 17 installed here. 8.13.2 removes that risk entirely; moving to 9.x later is a mechanical bump. |
| Kotlin | **2.4.20** | Latest stable. (2.5.0 exists only as Beta1 — not used.) |
| KSP | **2.3.12** | Room's annotation processor. KSP now versions independently of Kotlin — no `kotlin-version-KSP` suffix. KSP over kapt: required for the Room 2.8 KMP artifacts and considerably faster. |
| Java / Kotlin target | JVM **17**, via `jvmToolchain(17)` | Matches the installed JDK. |
| `compileSdk` / `targetSdk` | **36** | Only platform installed; matches the "Android 16 / API 36" requirement. |
| `minSdk` | **26** (proposed) | Brings `java.time` without core-library desugaring, adaptive icons, and the `PackageManager` query APIs the app picker needs. Costs ~1% of devices. See §8 Q6. |

### 2.2 Dependencies

| Dependency | Version | Purpose |
|---|---|---|
| `androidx.compose:compose-bom` | **2026.09.00** | Pins all Compose artifacts together. **Verified**: this BOM pins `material3` → **1.4.0** and `material3-window-size-class` → 1.4.0. |
| `androidx.compose.ui:ui`, `ui-graphics`, `ui-tooling-preview` | via BOM | UI + previews |
| `androidx.compose.material3:material3` | **1.4.0** (via BOM) | Material 3, incl. `MotionScheme` (see §6) |
| `androidx.compose.material3:material3-window-size-class` | 1.4.0 (via BOM) | Compact/medium sizing for the wallet grid |
| `androidx.compose.material:material-icons-extended` | 1.7.8 (via BOM) | Wallet/category icons — avoids an icon-font dependency |
| `androidx.activity:activity-compose` | 1.13.0 | `ComponentActivity`, `setContent`, `enableEdgeToEdge` |
| `androidx.lifecycle:lifecycle-runtime-compose`, `lifecycle-viewmodel-compose` | 2.11.0 | `collectAsStateWithLifecycle`, `ViewModel`, `viewModel()` |
| `androidx.navigation:navigation-compose` | 2.10.2 | Nav graph |
| `androidx.room:room-runtime` + `room-ktx` | **2.8.5** | Entities, DAOs, `Flow`; `room-compiler` supplied via KSP |
| `org.jetbrains.kotlinx:kotlinx-coroutines-android` | 1.11.0 | Transitive already, pinned explicitly |
| `junit`, `androidx.test.ext:junit` 1.3.0, Robolectric, `ui-test-junit4` | latest stable | **JVM/Robolectric only** — see §1 |

### 2.3 Gradle layout

- `settings.gradle.kts` — `pluginManagement` (`google`, `mavenCentral`, `gradlePluginPortal`), `dependencyResolutionManagement` with `repositoriesMode = FAIL_ON_PROJECT_REPOS`, module include `:app`.
- Root `build.gradle.kts` — plugins declared `apply false`.
- `gradle/libs.versions.toml` — **version catalog** as the single source of truth, so the table above is enforced by the build rather than by memory.
- `gradle.properties` — `android.useAndroidX=true`, `org.gradle.jvmargs=-Xmx3g`, `kotlin.code.style=official`, `android.nonTransitiveRClass=true`.
- `app/build.gradle.kts` — `buildFeatures { compose = true }`, KSP `room.schemaLocation`, `packaging { resources.excludes += "/META-INF/{AL2.0,LGPL2.1}" }`.
- **Single `:app` module for phase 1.** Multi-module (`:data` / `:domain` / `:ui`) is a phase-2 move, justified only once a second feature exists. The package boundaries in §3 keep the seams clean so that split is mechanical later.

---

## 3. Package & folder structure

Root package `com.<yourcompany>.budgettracker` (placeholder — see §8 Q1). Single module; layers expressed as folders.

```
app/src/main/java/com/x/budgettracker/
├── BudgetApp.kt                      # Application; owns the manual ServiceLocator
├── MainActivity.kt                   # edge-to-edge, setContent { BudgetNavHost() }
│
├── di/
│   └── AppContainer.kt               # lazy db, daos, repositories. No Hilt in phase 1.
│
├── data/
│   ├── local/
│   │   ├── BudgetDatabase.kt         # @Database(entities, version = 1, exportSchema = true)
│   │   ├── Converters.kt             # TransactionStatus <-> String
│   │   ├── WalletDao.kt
│   │   ├── CategoryDao.kt
│   │   ├── TransactionDao.kt         # includes the @Transaction confirm block (§5a)
│   │   └── relation/                 # projection POJOs — NOT entities
│   │       ├── WalletSummary.kt      # @Embedded wallet + pendingTotal + pendingCount
│   │       └── TransactionWithRefs.kt# @Embedded txn + walletName + categoryName
│   ├── entity/
│   │   ├── Wallet.kt
│   │   ├── Category.kt
│   │   ├── Transaction.kt
│   │   └── TransactionStatus.kt      # enum PENDING / CONFIRMED / CANCELLED
│   └── repository/
│       ├── TransactionRepository.kt  # the spec's "backend"; only class that touches DAOs
│       ├── WalletRepository.kt       # wallet + category CRUD (thin)
│       └── BudgetValidation.kt       # pure functions: remaining(), classify() — pure & unit-testable
│
├── domain/
│   └── model/
│       ├── CreateTransactionResult.kt   # Success / Warning(overBy) / Blocked(reason)
│       ├── ConfirmResult.kt             # Confirmed / OverBudget(overBy) / NotFound / AlreadySettled
│       └── PendingConfirmRequest.kt     # txnId, amount, walletName, appLabel, source
│
├── payments/
│   ├── InstalledAppsRepository.kt    # PackageManager enumeration -> label + packageName
│   └── PaymentAppLauncher.kt        # getLaunchIntentForPackage wrapper (spec §2)
│
├── confirm/
│   └── ConfirmCoordinator.kt        # StateFlow<PendingConfirmRequest?>; resume -> sheet trigger
│
├── navigation/
│   ├── BudgetNavHost.kt             # NavHost + routes + ConfirmSheet hosted here
│   └── Routes.kt
│
├── ui/
│   ├── screens/
│   │   ├── home/            HomeScreen.kt            HomeViewModel.kt
│   │   ├── newtransaction/  NewTransactionScreen.kt  NewTransactionViewModel.kt
│   │   ├── walletsetup/     WalletSetupScreen.kt     WalletSetupViewModel.kt
│   │   ├── categorysetup/   CategorySetupScreen.kt   CategorySetupViewModel.kt
│   │   └── pending/         PendingTransactionsScreen.kt  PendingTransactionsViewModel.kt
│   ├── components/          # reusable, screen-agnostic
│   │   ├── WalletCard.kt            AmountField.kt
│   │   ├── WalletDropdown.kt        CategoryDropdown.kt
│   │   ├── PendingBanner.kt         RemainingBudgetBar.kt
│   │   ├── ConfirmPaymentSheet.kt   AppPickerSheet.kt
│   │   ├── ValidationMessage.kt     AgeLabel.kt
│   │   └── MoneyText.kt             # the single ₹ formatting entry point
│   └── theme/
│       ├── Color.kt      # full tonal palette; light + dark ColorScheme (all roles, incl. surfaceContainer*)
│       ├── Type.kt       # M3 type scale
│       ├── Shape.kt      # 5-step M3 shape scale
│       ├── Motion.kt     # MotionScheme wiring (see §6)
│       └── Theme.kt      # BudgetTheme() — the ONLY place MaterialTheme is called
│
└── util/
    ├── Money.kt          # Double <-> paise-safe parse/format, locale-aware
    └── TimeFormat.kt     # epochMillis -> "2 days ago"
```

Resources and tests mirror this layout:
`res/values/{strings,themes}.xml`, `res/xml/{backup_rules,data_extraction_rules}.xml`,
`app/src/test/` for JVM/Robolectric tests, `app/src/androidTest/` left empty for now.

**Layering rules** (enforced by review, and by the absence of the corresponding imports):

- `ui/` → `domain/` + repositories → `data/`.
- `data/local` is reachable **only** from `data/repository`.
- No screen file imports `androidx.room.*`.
- No ViewModel imports a DAO. *(SPEC §4 permits VMs to call `WalletDao`; routing through `WalletRepository` keeps the same call sites with one fewer thing to leak.)*

**Dependency injection:** a manual `AppContainer` held by the `Application`, exposed via a
`viewModelFactory` passed to `viewModel()`. Hilt is deliberately **out** for phase 1 — there is
one screen graph and no scoping need beyond "singleton". Adding Hilt later is contained to
`AppContainer` plus one annotation per ViewModel.

---

## 4. Build order

Each step ends in a state that can be built, installed, and clicked through. No step depends on
an unfinished later step.

| # | Step | Independently testable when… |
|---|---|---|
| **0** | **Toolchain spike.** Gradle wrapper, `:app` with an empty `MainActivity`, pinned catalog versions. | `./gradlew :app:assembleDebug` succeeds → proves the AGP + Kotlin + KSP + SDK-36 combination resolves and downloads. De-risks the entire plan before any app code exists. |
| **1** | **Theme shell.** `ui/theme/*` plus a Home placeholder rendering "Hello" through `BudgetTheme`. | App launches; light/dark and dynamic colour visibly switch. Validates the Compose/M3 stack in isolation. |
| **2** | **Entities + DAOs + DB.** All three tables, `Converters`, indices, `exportSchema = true`, version 1. No repository yet. | Robolectric tests on an in-memory DB: insert/read/update per DAO; FK enforcement (a transaction with a bad `walletId` throws); indices asserted via `PRAGMA index_list`. |
| **3** | **Repository + validation.** `BudgetValidation` pure math first, then `TransactionRepository` including the `@Transaction` confirm block, plus `PaymentAppLauncher`. | JVM tests: table-driven budget math (limit/spent/pending permutations, zero-pending, over-limit, empty wallet); a confirm-atomicity test asserting `currentSpent` **and** row status change together or not at all. |
| **4** | **Navigation skeleton.** `BudgetNavHost`, routes, stub screens. | Back-stack navigation works between stubs; back stack survives process death. |
| **5** | **Home (read path).** `WalletSummary` aggregate query, `HomeViewModel`, `WalletCard`, empty state. | Real wallet rows render remaining/limit with a pending badge; the **query-count test** (§5c) proves N+1-free. |
| **6** | **Wallet + Category setup.** Full CRUD screens, delete/rename dialogs. | Create / edit / delete a wallet and a category; data survives an app restart. |
| **7** | **App picker.** `InstalledAppsRepository` enumeration + `AppPickerSheet`, wired into WalletSetup; package name persisted. | Picker lists the device's real installed apps; a chosen package persists; `PaymentAppLauncher` returns `true` for it and `false` for a bogus package. |
| **8** | **NewTransaction form.** Form state, live remaining-budget preview, validation messages, inline "+ new category", create PENDING. | An over-budget amount surfaces the warning live; submitting creates a PENDING row that immediately appears in Home's banner. |
| **9** | **Payment launch → resume → ConfirmSheet.** `ConfirmCoordinator`, saved-state marker, `ModalBottomSheet` at the NavHost root. | **The hard one** — verified against the case matrix in §7: rotation mid-flow, back press, screen off/on, failed launch, double resume. |
| **10** | **PendingTransactions screen.** Joined query with wallet + category names, age labels, per-row confirm/cancel. | A 3-day-old pending row reads "3 days ago"; cancel leaves `currentSpent` untouched; confirm updates Home instantly. |
| **11** | **Empty states, a11y, polish.** Content descriptions, TalkBack labels on the amount/progress bar, IME action wiring, landscape. | Manual pass, plus `lintDebug` clean. |
| **12** | **Packaging decisions.** Backup / data-extraction rules, R8 baseline, release build type. | `assembleRelease` succeeds and installs; force-stop → relaunch preserves the ledger. |

**Phase-2 hooks land in step 9, not earlier.** `PendingConfirmRequest.source` is an enum
(`MANUAL_LAUNCH` today; `QR_SCAN` and `UPI_DEEP_LINK` reserved), and `ConfirmCoordinator` exposes
`onExternalPaymentDetected(request)` — the same entry point `onNewIntent` will call in phase 2.
Nothing in phase 1 needs to change when QR scanning arrives.

---

## 5. The three correctness guarantees

### (a) Confirm is a single atomic Room `@Transaction`

**Mechanism** — a `@Transaction`-annotated `suspend` function on `TransactionDao`, *not* a
multi-call sequence in the repository. Room generates code that wraps the body in a real SQLite
transaction, and Room's `beginTransaction` issues `BEGIN IMMEDIATE`, so the write lock is taken
**before the first read**. No other writer can interleave between the validation read and the
balance write.

The body, in order:

1. `SELECT` the transaction row by id.
2. If missing or not `PENDING`, return without writing — makes a double-tap confirm, or a confirm
   after cancel, an **idempotent no-op** rather than a double-spend.
3. `SELECT` the wallet by `walletId`.
4. `SELECT COALESCE(SUM(amount), 0) FROM transactions WHERE walletId = :id AND status = 'PENDING'`.
5. Budget check against `limit − currentSpent − pendingTotal` (§5b).
6. `UPDATE transactions SET status = 'CONFIRMED', confirmedAt = :now WHERE id = :id`.
7. `UPDATE wallets SET currentSpent = :recomputedSpent WHERE id = :walletId`.

**Key design choice:** step 7 writes a **recomputed** value —
`SELECT SUM(amount) FROM transactions WHERE walletId = :id AND status = 'CONFIRMED'` — never an
increment. The aggregate is re-derived from the source of truth *inside the same transaction*, so
`currentSpent` is structurally incapable of drifting from reality. Cost: one indexed `SUM` per
confirm, negligible for a single-user ledger. Increment-only arithmetic is the classic source of
drift bugs; recomputing removes that entire bug class.

**Belt and braces:** the repository invokes this DAO function from a single-threaded dispatcher,
and `ConfirmCoordinator` disables the sheet's action buttons while a confirm is in flight — so two
concurrent confirms are impossible from the UI even before the DB lock is considered.

**Deliberately not using SQLite triggers.** A trigger could enforce the invariant at the DB level,
but Room cannot unit-test trigger logic, cannot surface validation failures through a DAO return
value, and it makes the schema opaque to KSP validation. A Kotlin-side `@Transaction` plus a
recomputed aggregate yields the same atomicity *plus* typed errors. Noted as a conscious trade-off.

**Test that proves it:** a Robolectric test calls `confirmTransaction` concurrently from N
coroutines against N distinct pending rows, then asserts the final `currentSpent` equals the sum
of exactly N amounts and that exactly N rows are `CONFIRMED`.

### (b) Budget validation counts PENDING **and** CONFIRMED — at create *and* at confirm

One definition, used by both paths, expressed as a single SQL projection so it cannot drift
between them:

```
remaining(w) = w.budgetLimit
             − w.currentSpent
             − COALESCE((SELECT SUM(amount) FROM transactions
                           WHERE walletId = w.id AND status = 'PENDING'), 0)
```

`currentSpent` already means *CONFIRMED only* (entity contract, SPEC §1), so
`currentSpent + pendingSum` is the full committed-plus-reserved exposure. `COALESCE` handles the
no-pending case; a bare `SUM` over an indexed predicate returns `NULL` there, which is the classic
bug this guards against.

- **At create** — the validation query and `INSERT … status = 'PENDING'` execute in **one**
  `@Transaction` (`insertPendingWithValidation`). Validating in a repository method and inserting
  separately would leave a window in which a concurrent confirm shrinks the budget between the two
  statements. Wrapping it makes create-time validation exactly as strong as confirm-time.
- **At confirm** — steps 3–5 of §5a: same query, same transaction as the writes.
- **In the UI** — `NewTransactionViewModel` reads the same projection through a `Flow`
  (`observeRemainingBudget(walletId)`) to drive the live preview, so the number the user sees and
  the number the database enforces come from one implementation. The preview is advisory only; the
  authoritative check is always the transactional one.
- **Classified, not boolean** — the result is `WithinBudget` / `ExceedsBy(x)` / `Blocked(reason)`,
  so the UI can render three states and one implementation serves both the create-time warning and
  the confirm-time overage notice.
- **Form-level `amount > 0`** is enforced by the `AmountField` parse (rejecting empty, `0`,
  negative, and >2-decimal junk) **and** re-checked inside `insertPendingWithValidation` — the
  form is never trusted.

**Tests:** a table-driven suite over `{limit, spent, pending[], newAmount}` asserting
`WithinBudget` / `ExceedsBy`; plus an interleaving test proving that create-time validation sees a
confirm that lands a microsecond earlier.

### (c) No N+1 queries — named mechanism, not vigilance

**The rule:** every list screen is served by **one** query returning a purpose-built POJO. No
screen enriches rows inside a `for` loop, and no DAO method is ever called from a list-item
composable.

| Screen | Single query | Shape |
|---|---|---|
| Home | `observeWalletSummaries(): Flow<List<WalletSummary>>` | `Wallet` LEFT JOIN a grouped sub-select over `transactions` producing `pendingTotal` and `pendingCount`. One row per wallet, `GROUP BY w.id`. Room invalidates and re-emits the whole list on any wallet or transaction write. |
| PendingTransactions | `observePendingWithRefs(): Flow<List<TransactionWithRefs>>` | One INNER JOIN `transactions → wallets`, LEFT JOIN `categories`, selecting `w.name AS walletName` and `c.name AS categoryName` as POJO columns. Wallet and category names arrive **already joined**; the composable performs no lookups. |
| Wallet history | `observeTransactionsForWallet(id)` | Same join shape, `WHERE t.walletId = :id`. |

`WalletSummary` and `TransactionWithRefs` live in `data/local/relation/` and are **not**
`@Entity` — they are projection POJOs, so Room emits a schema warning if a join shape is wrong,
and they cannot be written by accident.

**Indices** — declared in `@Entity(indices = [...])`; Room additionally auto-creates indices for
foreign-key columns:

- `transactions(walletId, status)` — serves the per-wallet PENDING sum, the per-wallet history,
  and the join key.
- `transactions(status, createdAt DESC)` — serves `getAllPending()` and the age ordering on
  PendingTransactions.
- `wallets(name)` for wallet dropdown ordering; `categories(name)` for category dropdown ordering.

**How this is enforced rather than merely intended:** `RoomDatabase.setQueryCallback` is attached
in the test source set. A Robolectric test exercises each screen's data path, counts executed
statements, and asserts the count is **constant** (1–2) whether the list holds 1 row or 200.
Inserting 200 wallets and transactions and re-running means that if anyone reintroduces a loop,
the count scales with N and the test fails. This is a permanent regression guard, not a code
review convention.

---

## 6. Material 3 Expressive theming — what actually exists

I inspected the shipped bytecode rather than trusting memory. Compose BOM **2026.09.00** pins
**material3 1.4.0**.

**Verified present in material3 1.4.0 stable:**

- `ColorScheme` (full role set), `Typography`, `Shapes`, `MaterialTheme` — baseline M3 theming.
- **`MotionScheme`**, plus `StandardMotionTokens` and **`ExpressiveMotionTokens`** — real motion
  theming is available on stable.
- The `ExperimentalMaterial3ExpressiveApi` marker annotation.
- `AppBarColumn` / `AppBarRow` (Expressive-family app bar layouts).

**Verified NOT present in material3 1.4.0 stable:** `LoadingIndicator` /
`WavyLoadingIndicator`, `ButtonGroup`, `SplitButton`, `FloatingToolbar`,
`ContainerWithBackground`, and `MaterialShapes` (the tap-to-morph shape system).

I also confirmed that the separate `androidx.compose.material3:material3-expressive` artifact
**no longer exists** (HTTP 404) — it was folded into `material3`. Whether these components have
landed in the `1.5.0-alpha29` line is **unverified** (that artifact was not downloaded), so treat
them as *unavailable on stable, possibly available on alpha*.

**Explicit substitutions for phase 1 (stable only):**

| M3 Expressive capability | Stable substitute |
|---|---|
| Wavy loading indicators | `CircularProgressIndicator` / `LinearProgressIndicator` driven by `MotionScheme.expressive()` springs, so motion still reads as expressive |
| `ButtonGroup` | `Row` with `weight(1f)` on `FilledTonalButton`s, or `FlowRow` when actions must wrap |
| `SplitButton` | A `FilledTonalButton` plus a small `IconButton` in a `Row` sharing `Shape` and border |
| `FloatingToolbar` | `LargeTopAppBar` / `BottomAppBar` (both stable) |
| `MaterialShapes` shape morphing | The static `Shapes` scale; no press-to-morph on the FAB |
| Additional expressive colour roles | Derived in `Color.kt` from the baseline `ColorScheme` rather than read from the library |

**Theme wiring:** `ui/theme/Theme.kt` is the single `MaterialTheme` call site.
`BudgetTheme(darkTheme, dynamicColor)` applies dynamic colour on API 31+ (minSdk 26, so gated)
and falls back to the hand-built brand `ColorScheme`; `Motion.kt` supplies
`MotionScheme.expressive()` with a fallback to `standard()`; `Type.kt` and `Shape.kt` are plain M3
scales. Because everything routes through one `BudgetTheme` composable, adopting material3 1.5
alphas later — or swapping in the genuine Expressive components — is a **one-file change plus a
version-catalog bump**, which is precisely the property that keeps phase 2 cheap.

Recommendation: ship stable and be slightly less expressive, rather than freezing the app on an
alpha channel. Easy to reverse if you would rather take the alpha.

---

## 7. `onResume` → ConfirmSheet: making it reliable

The naive version — fire a sheet whenever the activity resumes — breaks in five ways: cold start,
rotation, returning from an unrelated dialog, screen off/on, and double resume. The fix is a
**one-shot launch marker**, not a resume check.

**`ConfirmCoordinator`** — a singleton `StateFlow<PendingConfirmRequest?>` owned by an
Activity-scoped ViewModel at the NavHost root:

1. **Arm.** `NewTransactionScreen` taps "Open [App]" → `TransactionRepository.createTransaction(…)`
   returns the new PENDING row → the coordinator maps it to a
   `PendingConfirmRequest(txnId, amount, walletName, appLabel, source = MANUAL_LAUNCH)` and calls
   `arm(request)`, which writes the marker into **`SavedStateHandle`** (survives process death,
   unlike a plain field) → then `startActivity`.
2. **Observe.** `MainActivity` registers a `LifecycleEventObserver` for `ON_PAUSE` / `ON_RESUME`
   (or `LifecycleResumeEffect` in Compose). It **never** decides anything on resume by itself.
3. **Discharge.** On `ON_RESUME`, the coordinator shows the sheet **only if all three guards
   hold**:
   - a marker exists — one-shot: it is consumed and cleared in the same state update, so a second
     resume is a no-op;
   - an `ON_PAUSE` was actually observed since arming — rules out cold start and rotation-only
     resumes;
   - `SystemClock.elapsedRealtime() − armedAt ≥ ~800 ms` — rules out an instant bounce, a
     screen-off flicker, and a failed launch.
4. **Decide.** Sheet actions call `confirmTransaction` / `cancelTransaction` / `editAmount`, each
   clearing the marker. An `ActivityNotFoundException` from the launcher clears the marker and
   surfaces a snackbar, so a missing app cannot leave a phantom armed state.

**Why `SavedStateHandle` plus a single nullable state value:** the invariant is "at most one armed
request, consumed exactly once". Rotation recreates the Activity, but the ViewModel survives and the
marker lives in saved state — so you can neither get two sheets nor lose one. The marker is cleared
*before* the sheet is shown, not after, so re-entering the app later cannot resurrect a stale one.

**Sheet placement:** `ConfirmPaymentSheet` is hosted **outside the `NavHost`**, as a sibling in the
`BudgetNavHost` layout. It is attached to no back-stack entry, so it survives the
NewTransaction → Home transition and cannot be lost to a `popBackStack()`.

**Dismissal semantics:** the `ModalBottomSheet`'s `onDismissRequest` (swipe-down or back) does
**not** silently discard the transaction — it routes to the same handler as Cancel, leaving the row
`PENDING` and showing "Kept as pending." Swiping away must never mean "the money wasn't spent."
*This needs sign-off — see §8 Q5.*

**Phase-2 fit:** `onNewIntent` (for `upi://pay`) and a future ML Kit QR callback both call the same
`onExternalPaymentDetected(request)` with `source = UPI_DEEP_LINK` / `QR_SCAN`. Zero changes to the
sheet, the coordinator's discharge logic, or the ViewModels.

---

## 8. Open questions & decisions needed before code

### Blocking — I need answers before writing code

1. **Application ID / package name.** I need a real one (`com.yourcompany.budgettracker`?) plus the
   user-visible app name. Every path in §3 depends on it.
2. **Monthly reset — the biggest gap in the spec.** `budgetLimit` is documented as a *monthly*
   limit, but `Wallet` has no month or period column and `currentSpent` never resets. As specced,
   `currentSpent` accumulates forever and every wallet trends to permanently over-budget. Pick one:
   - **(a)** Add `budgetPeriodStart: Long` to `Wallet`; `currentSpent` means "spent since
     `budgetPeriodStart`"; the repository lazily rolls the period over on first access after a month
     change. *(Recommended — one defaulted column, near-zero cost now, versus a data migration plus
     a semantic change later.)*
   - (b) Treat it as a lifetime budget (and rename the field to match).
   - (c) A manual "Start new month" button.
3. **Over-budget policy.** SPEC §5.2 says "your call: hard block vs. warn." I recommend
   **warn-and-continue at create** (a confirm dialog showing "over by ₹X") and **always allow at
   confirm** — refusing to record a payment the user demonstrably made is worse than showing an
   overage. Confirm?
4. **`linkedAppPackage` nullability.** The spec types it as non-null `String`, but nothing forces
   the user to pick an app during setup. I recommend nullable, an explicit "No app linked" state,
   and a disabled "Open [App]" button showing "app not installed, update wallet settings"
   (SPEC §5.4). Alternatives: a `""` sentinel, or forcing a pick. Your call.
5. **ConfirmSheet dismissal** — treat swipe-away as *keep PENDING* (recommended) or as *Cancel*?
6. **minSdk 26** — acceptable to drop pre-2018 devices?

### Non-blocking — I'll assume these defaults and note them in code unless you object

7. **App picker scope.** There is no reliable "is this a UPI app" API. Plan: enumerate all
   launchable apps (`MAIN` / `LAUNCHER` via `queryIntentActivities`), show each with its real
   launcher label and icon, sorted with a heuristic boost for known payment packages
   (PhonePe / GPay / Paytm / …) **and** for apps resolving an `upi://` intent filter, plus a search
   field. Fully accurate UPI detection is not possible — accept the heuristic?
8. **Gaps in the spec's DAO tables.** They omit `deleteWallet`, `updateCategory` (rename), and any
   category reassignment. I will add exactly those three, nothing more.
9. **FK delete behaviour.** Deleting a wallet cascades its transactions (behind a confirmation
   dialog showing the count). Deleting a category sets `Transaction.categoryId = NULL` rather than
   cascading — deleting a tag should not destroy financial records. Confirm?
10. **Money as `Double`.** The spec mandates it. Floating-point money will eventually produce
    `₹99.99999999`. Phase 1 will follow the spec exactly, but every conversion is isolated in
    `util/Money.kt` (parse and format in one place, `.roundToInt()` to paise on write) so a later
    `Long`-paise migration touches a single file. Flagging the risk; not deviating from the spec.
11. **Currency / locale.** The spec hardcodes `₹`. Plan: format via
    `NumberFormat.getCurrencyInstance()` on the device locale for now — which would show `$` on a
    US phone, arguably a bug for an INR app — with `Money.kt` as the single swap point. Prefer a
    hardcoded INR locale instead?
12. **Testing strategy.** No emulator or system images are installed here, so `androidTest` can be
    written but never run. Plan: **JVM + Robolectric only** (in-memory Room; Compose UI tests under
    Robolectric) as the verification backbone. Acceptable — or should I attempt to install a system
    image (large download, and probably no KVM in this container)?
13. **AGP 8.13.2 rather than the newer 9.4.1** — chosen for JDK 17 compatibility. Fine, or should I
    chase 9.x (which may require installing a newer JDK)?
14. **Backups.** SPEC §7 flags the lack of backup as a risk. I'd enable `allowBackup` with explicit
    data-extraction / backup rules covering the Room DB — near-free insurance. In, or deliberately
    out for v1?
15. **Release build** — debug-only for phase 1, or set up a release signing config and R8 rules now?
16. **First-run state** — ship with an empty Home plus a "create your first wallet" CTA, or seed a
    demo wallet and categories? (I'd say empty: no demo data in a finance app.)

---

## 9. On approval — what happens next

Work starts at **Step 0** (toolchain spike) and proceeds down the §4 order, reporting back at each
step's testability checkpoint. Work pauses for consultation before **Step 9** (the resume /
confirm-sheet wiring), since that is the only step carrying real design risk and the one your Q5
answer changes.

No application code will be written until you explicitly approve this plan.