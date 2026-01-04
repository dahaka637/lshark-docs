# Contract Lifecycle — Creation & Automatic Collection

This document consolidates the LSHARK contract lifecycle into a single, cohesive explanation covering:
- **Part 1:** Contract creation, delivery, and acceptance
- **Part 2:** Automatic debt collection (auto-collection), including partial payments, fines, and caps

The lifecycle is designed around **server authority**, **explicit debtor consent**, and **auditable state transitions**.

---

## Part 1 — Creation, Delivery, and Acceptance

### 1) Proposal Creation (Creditor)

A contract begins as a **proposal** initiated by the creditor.

**Manual contracts**
- Created from the lender UI (Management tab).
- The client builds a proposal and sends it to the server.

**Auto-loans**
- Initiated via the server-side auto-loan subsystem.
- Auto-loans are validated against the creditor configuration and then issued through the same contract manager path.

---

### 2) Server Reception & Validation (Authority)

On manual proposal submission, the server receives:

- **Net message:** `LSHARK_Contract_Send`

**Manual payload (conceptual order)**

```txt
LSHARK_Contract_Send
├─ code (string)
├─ creditor_name (string)        -- display-only, server re-validates
├─ debtor_name (string)          -- used to resolve target player
├─ amount (float)                -- principal
├─ interest (float)              -- interest percent
├─ fine (float)                  -- fine percent
├─ total_with_interest (float)   -- client preview (server still validates)
└─ installments (uint16)
```

The server is authoritative and validates:
- creditor identity and job permission
- debtor resolution (must be online and valid)
- self-contract restrictions (unless debug)
- job limits (interest, fine, installments, multiplier cap)
- creditor internal balance availability

If any check fails, the proposal is rejected and the creditor is notified.

---

### 3) Pending Staging (No Persistence Yet)

Approved proposals are staged **in memory** as pending contracts:

- `PENDING_CONTRACTS[code] = contractTable`

This ensures:
- no contract is persisted without debtor consent
- rejected/expired offers are discarded without touching storage

Pending states typically include:
- `pending` for manual proposals
- `auto_pending` for auto-loans
- `origin` metadata to track issuance source

---

### 4) Delivery to Debtor (Server → Client)

The server sends the offer to the debtor:

- **Net message:** `LSHARK_Contract_Receive`

**Delivery payload (conceptual)**

```txt
LSHARK_Contract_Receive
├─ code (string)
├─ creditor (string)
├─ creditor_steamid (string)
├─ debtor (string)
├─ debtor_steamid (string)
├─ amount (float)
├─ interest (float)
├─ fine (float)
├─ installments (uint16)
└─ total_with_interest (float)
```

This payload is intentionally minimal and UI-oriented.

---

### 5) Debtor Decision (Accept / Reject)

On the client:
- Auto-loans (`AUTO-...`) open the signing UI immediately.
- Manual offers show a notification with quick actions (open/accept or reject).

The debtor responds with one of:

- `LSHARK_Contract_Accept` (payload: `code`)
- `LSHARK_Contract_Reject` (payload: `code`)

---

### 6) Acceptance Finalization (Server)

When the debtor accepts, the server:

1. validates pending existence and debtor ownership
2. re-checks creditor balance availability
3. transfers money:
   - deduct principal from creditor internal balance
   - credit principal to debtor DarkRP wallet
4. persists the contract:
   - `status = "active"`
   - `LSHARK.DB_InsertContract(contractTable)`
5. emits:
   - `hook.Run("LSHARK_ContractCreated", contractTable)`
6. registers:
   - logs (audit)
   - analytics (timeline)
7. notifies both parties
8. removes the pending record

Rejection simply removes the pending record and notifies the creditor, with **no persistence** and **no transfer**.

---

## Part 2 — Automatic Debt Collection (Auto-Collect)

Automatic collection is the enforcement stage that processes active contracts over time.  
It is fully server-authoritative and runs periodically for **online debtors**.

---

Before each collection cycle, LSHARK gathers every contract where the online player is the debtor and the contract is eligible for enforcement (`active` or `capped`). For each contract, the server first normalizes the financial state (remaining balance, total paid, installment counters) and applies protective rules that keep the system stable over time.

The first protection is the **debt ceiling** (multiplier cap). If the contract’s remaining balance has grown beyond the maximum allowed for the creditor job, LSHARK clamps the remaining balance to that ceiling, switches the contract into `capped`, and disables late fees (`fine = 0`). From that point forward, the contract can still be paid down, but it can no longer grow due to penalties.

Once the contract is inside allowed bounds, the server determines what the debtor is allowed to pay automatically in this tick by computing the **auto-debit cap** from the debtor’s current DarkRP wallet balance and their configured percentage limit. This cap is a hard limit for the tick: LSHARK will not auto-debit more than that amount, even if the contract is behind.

With the cap established, LSHARK computes the **current installment target** based on the remaining balance divided by the number of installments still open. Then it processes one of the three payment outcomes. If the debtor has enough available cap to cover the target installment, the server debits the debtor’s DarkRP wallet and credits the same value into the creditor’s internal LSHARK balance, advances the installment counter, reduces the remaining balance, and if the remaining reaches zero it marks the contract as `paid`. If the debtor can pay something but not the full target installment, the server still debits what it can (up to the cap), credits the creditor’s LSHARK balance, reduces the remaining balance, and then evaluates penalty rules.

Late fees are applied only when the contract is not capped and the fine percentage is greater than zero. They occur when the debtor cannot meet the expected payment for the tick (for example: no available cap, no wallet money, or only a partial payment). In those cases, LSHARK increases the remaining balance by a fine computed from the remaining amount under the configured fine rate, persists the updated remaining balance, and emits the corresponding hooks/logs/analytics. If the contract is `capped`, the same failure or partial-payment scenarios do not apply any fine; the system only records the event and continues.

All money movement is strictly server-side and always follows the same direction: payments are taken from the debtor’s DarkRP wallet and deposited into the creditor’s LSHARK internal balance. No client-side calculation or UI action can force these transitions; the client only observes the results through updates, notifications, and events.

---

### 1) Timer & Scheduling

The engine runs on a repeating timer:

- `LSHARK_AutoCollectTimer`

It is started (and restarted) on:
- `InitPostEntity`
- `PostCleanupMap`

Its interval is configured by:

- `Settings.General.AutoDebtProcessingInterval`

---

### 2) Eligibility Rules

On each cycle, the server processes contracts for each online player where:
- `debtor_steamid == ply:SteamID()`

Eligible statuses:
- `active`
- `capped`

Skipped:
- `paused`
- `paid`

---

### 3) Debtor Auto-Debit Limit (Cap)

Before debiting money, the engine computes the debtor cap:

1. debtor wallet:
   - `wallet = ply:getDarkRPVar("money")`
2. percent limit:
   - debtor-specific limit if stored
   - otherwise fallback to `Settings.General.DefaultDebtDiscount`
3. final debit cap:
   - `debitCap = floor(wallet * (pct / 100))`

This prevents auto-collection from consuming more than the debtor configured as eligible.

---

### 4) Installment Target Computation

For an eligible contract, installment target is computed from remaining balance:

```txt
installmentsRemaining = max(installments - installments_paid, 1)
targetInstallment     = remaining_balance / installmentsRemaining
```

This keeps installments adaptive as remaining balance changes (including fines and partials).

---

### 5) Debt Cap Enforcement (Multiplier Ceiling)

Before fines and before progressing, the engine ensures the contract does not exceed the maximum debt multiplier:

- `cap = amount * MaxLoanMultiplier` (from job limits)

If the contract exceeds cap:
- remaining is clamped to cap
- fine is set to `0` (late fee disabled)
- status becomes `capped`
- the contract is recorded as limited
- hooks/logs/analytics are emitted

This prevents infinite debt escalation.

---

### 6) Processing Outcomes

The engine chooses one of the following paths:

#### A) No money available (failure path)

If `wallet <= 0` or `debitCap <= 0`:
- if status is `capped`: no fine, just notify/log
- if status is `active` and fine > 0: apply late fee on remaining balance and persist
- if status is `active` and fine == 0: persist unchanged, notify/log

#### B) Full installment payment

If `debitCap >= targetInstallment`:
- debit installment value
- credit creditor internal balance
- increment `installments_paid`
- reduce `remaining_balance`
- if settled: set `status = "paid"`, set remaining to 0, emit payoff hook

Hooks/logs/analytics and notifications are emitted for the installment payment.

#### C) Partial payment

If `debitCap > 0` but `< targetInstallment`:
- debit partial amount
- credit creditor internal balance
- reduce remaining balance by partial
- if active and fine > 0: apply fine on the new remaining
- if capped: no fine

Persist, notify, and emit logs/analytics accordingly.

#### D) Advanced installment consumption (if present)

If `installments_advanced > 0`, the engine consumes one advanced installment for that tick:
- decreases `installments_advanced`
- persists updated value
- emits logs/analytics
- no money transfer and no fine

---

### 7) Persistence & Consistency

Every enforcement tick persists a consistent snapshot using the contract storage layer.  
Updates typically include:
- `total_paid`
- `remaining_balance`
- `installments_paid`
- `installments_advanced`
- `fine` (possibly disabled when capped)
- `status` transitions: `active → capped`, `active/capped → paid`

This ensures restarts do not break progression and all state remains durable.

---

### 8) Auditability & Extensibility

Auto-collection emits:
- notifications to debtors and (when online) creditors
- logs for every major event
- analytics events for the dashboard timeline
- lifecycle hooks for third-party integrations

Common hooks include:
- `LSHARK_LimitedContract(code)`
- `LSHARK_FineApplied(code)`
- `LSHARK_InstallmentPaid(code)`
- `LSHARK_ContractPaidOff(code)`

---

## Summary

The LSHARK lifecycle is built to guarantee:
- **server authority** over every state transition
- **debtor consent** before persistence and money transfer
- **bounded enforcement** via debtor debit caps and multiplier ceilings
- **durable persistence** on every meaningful update
- **full observability** through notifications, logs, analytics, and hooks
