# Contract Model & Database Structures

This document describes the **LSHARK contract model** (what a contract is, its fields, and how they are interpreted), and then documents the **SQLite tables** used by LSHARK.

---

## Contract Model

In LSHARK, a **contract** is the authoritative server record representing a loan agreement between a **creditor** and a **debtor**, including its terms (principal, interest, fine, installments) and its evolving state (paid amounts, remaining balance, status).

Contracts are persisted in SQLite in the `LSHARK_CONTRACTS` table and are treated as the single source of truth for debt enforcement, UI displays, logs, and analytics.

### Contract Status Values

A contract’s `status` represents its current operational state:

- **active**: normal contract; installments and fines may apply.
- **paused**: temporarily disabled; typically excluded from automatic processing.
- **paid**: fully settled; remaining balance is zero.
- **capped**: debt growth has reached the configured multiplier ceiling; additional fines are typically disabled and the debt is clamped.

---

## Contract Structure (Raw)

### SQLite Row Schema: `LSHARK_CONTRACTS`

```txt
LSHARK_CONTRACTS
├─ code (TEXT, PRIMARY KEY)
├─ creditor (TEXT)
├─ creditor_steamid (TEXT)
├─ debtor (TEXT)
├─ debtor_steamid (TEXT)
├─ creditor_job (TEXT)
├─ amount (REAL DEFAULT 0)
├─ interest (REAL DEFAULT 0)
├─ fine (REAL DEFAULT 0)
├─ installments (INTEGER DEFAULT 1)
├─ installments_paid (INTEGER DEFAULT 0)
├─ installments_advanced (INTEGER DEFAULT 0)
├─ total_with_interest (REAL DEFAULT 0)
├─ total_paid (REAL DEFAULT 0)
├─ remaining_balance (REAL DEFAULT 0)
├─ status (TEXT DEFAULT 'active')
└─ created_at (TEXT)
```

### Field Semantics

#### Identity

- **code**  
  Unique contract identifier (primary key). Used across the entire system as the stable reference for payments, logs, analytics, and admin actions.

#### Parties

- **creditor**, **creditor_steamid**  
  Creditor display name and SteamID. SteamID is the authoritative identifier; the name is for display/auditing.

- **debtor**, **debtor_steamid**  
  Debtor display name and SteamID. SteamID is used for ownership validation, queries, and enforcement.

- **creditor_job**  
  Job name captured at creation time. Used for policy context (limits and multiplier rules) and for audit visibility.

#### Financial Terms

- **amount**  
  Principal (original loaned value).

- **interest**  
  Interest percentage applied to the loan (stored as a percentage value). Depending on the contract type, this may already represent the *total* interest for the full contract.

- **fine**  
  Late fee percentage applied on missed/partial installment processing, usually calculated on top of the **remaining balance**.

- **installments**  
  Total number of installments agreed in the contract.

#### Accounting & Progress

- **total_with_interest**  
  Total debt value after applying interest to the principal (baseline total before late fees).

- **total_paid**  
  Total amount already paid into this contract (sum of all payments recorded by LSHARK).

- **remaining_balance**  
  Current unpaid balance. Decreases with payments, may increase with fines until capped by the multiplier rule, and becomes `0` when the contract is fully settled.

- **installments_paid**  
  Count of installments successfully paid so far. Used for progress tracking and status transitions.

- **installments_advanced**  
  Tracks installments treated as “advanced/covered” by the payment logic. This value must not exceed `installments_paid`, and is used to keep installment progression consistent when special installment handling exists.

#### State & Metadata

- **status**  
  Operational state string (see status values above).

- **created_at**  
  Creation timestamp stored as text (used for auditing and UI display).

---

## Contract Structure (Example)

### Example Contract Row (Conceptual)

```lua
{
  code = "AUTO-123456",
  creditor = "Banker Bob",
  creditor_steamid = "STEAM_0:1:11111111",
  debtor = "Player Alice",
  debtor_steamid = "STEAM_0:0:22222222",
  creditor_job = "Banker",
  amount = 5000,
  interest = 40,
  fine = 2,
  installments = 10,
  installments_paid = 3,
  installments_advanced = 3,
  total_with_interest = 7000,
  total_paid = 2100,
  remaining_balance = 4900,
  status = "active",
  created_at = "2026-01-03 21:00:00"
}
```

---

## Related Database Structures

LSHARK uses additional tables for internal wallets, logs, analytics, auto-loan settings, debtor limits, and capped contract tracking.

### 1) Lender Balance Wallet: `LSHARK_BALANCE`

Stores an internal lender wallet per SteamID (separate from DarkRP cash).

```txt
LSHARK_BALANCE
├─ steamid (TEXT, PRIMARY KEY)
└─ balance (REAL DEFAULT 0)
```

**Purpose:**  
- Holds lender funds used to issue loans and receive repayments.
- Supports deposit/withdraw flows between DarkRP money and the internal balance.

---

### 2) Audit Logs: `LSHARK_LOGS`

Stores server-generated log entries for transparency and debugging.

```txt
LSHARK_LOGS
├─ id (INTEGER, PRIMARY KEY AUTOINCREMENT)
├─ steamid (TEXT)
├─ contract_code (TEXT)
├─ type (TEXT)
├─ message (TEXT)
└─ timestamp (TEXT)
```

**Purpose:**  
- Keeps a chronological history of actions (payments, fines, contract changes, admin actions).
- Feeds the client “Activity Logs” tab.

**Notes:**  
- `contract_code` links logs back to `LSHARK_CONTRACTS.code` when applicable.
- Cleanup routines may prune old logs and remove orphaned rows when contracts are missing.

---

### 3) Analytics Events: `LSHARK_ANALYTICS`

Stores event-based financial tracking used by the analytics dashboard.

```txt
LSHARK_ANALYTICS
├─ id (INTEGER, PRIMARY KEY AUTOINCREMENT)
├─ steamid (TEXT)
├─ contract_code (TEXT)
├─ event_type (TEXT)
├─ amount_loaned (REAL DEFAULT 0)
├─ amount_received (REAL DEFAULT 0)
├─ open_balance (REAL DEFAULT 0)
└─ timestamp (INTEGER)
```

**Purpose:**  
- Records events such as contract creation, payments received, and open balance changes.
- Feeds time-series charts and KPI aggregation client-side.

**Notes:**  
- `timestamp` is stored as Unix time (INTEGER) for ordering and charting.
- Cleanup routines may limit the number of tracked contracts per player and remove orphaned analytics.

---

### 4) Auto-Loan Configuration: `LSHARK_AUTOLEND`

Stores per-creditor auto-loan settings.

```txt
LSHARK_AUTOLEND
├─ steamid (TEXT, PRIMARY KEY)
├─ max_loan (REAL DEFAULT 0)
├─ max_installments (INTEGER DEFAULT 1)
├─ penalty_rate (REAL DEFAULT 0)
└─ interest_rate (REAL DEFAULT 0)
```

**Purpose:**  
- Defines automatic loan constraints per lender (credit limit, installment cap, rates).
- Used by the auto-loan request/validation flow server-side and for client preview UI.

---

### 5) Debtor Auto-Debit Limit: `LSHARK_DEBTOR_LIMITS`

Stores per-debtor automatic debit configuration.

```txt
LSHARK_DEBTOR_LIMITS
├─ id (INTEGER, PRIMARY KEY AUTOINCREMENT)
├─ steamid (TEXT NOT NULL UNIQUE)
├─ balance_limit (INTEGER DEFAULT 100)
└─ updated_at (TEXT)
```

**Purpose:**  
- Controls how much of a debtor’s available balance is eligible for automatic collection.
- Read/write is server-authoritative; UI only requests/updates via network messages.

---

### 6) Capped Contract Registry: `LSHARK_LIMITED_CONTRACTS`

Tracks contracts that have been marked as limited/capped.

```txt
LSHARK_LIMITED_CONTRACTS
├─ contract_code (TEXT, PRIMARY KEY)
└─ created_at (TEXT)
```

**Purpose:**  
- Maintains a durable record of contracts that reached the multiplier ceiling.
- Allows consistent enforcement and auditing of capped behavior across restarts.

---

## Indexes (Performance)

LSHARK creates indexes for common queries:

- Contract lookups by creditor SteamID and debtor SteamID
- Log lookups by steamid and contract_code
- Analytics lookups by steamid
- Limited-contract lookups by contract_code

These indexes keep contract retrieval, UI refreshes, and periodic maintenance efficient even with large datasets.

---

## Implementation Notes

- LSHARK treats SteamIDs as the stable identity keys for parties and balance owners.
- Contracts, logs, and analytics are designed so that **contract_code** links related records across tables.
- Clients never write directly to persistence; all mutations occur server-side and are validated before being saved.
