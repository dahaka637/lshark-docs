# API Overview

---

## Contract Structure (Authoritative Model)

A contract is the central authoritative object in LSHARK.  
It represents a loan agreement between a **creditor** and a **debtor**, fully enforced server-side.

### Core Fields

```txt
Contract
├─ code                     -- unique contract identifier
├─ creditor                 -- display name
├─ creditor_steamid          -- stable owner identifier
├─ debtor                   -- display name
├─ debtor_steamid            -- stable beneficiary identifier
├─ creditor_job              -- job at creation time
├─ amount                   -- principal
├─ interest                 -- interest percentage
├─ fine                     -- late fee percentage
├─ installments             -- total installments
├─ installments_paid        -- completed installments
├─ installments_advanced    -- prepaid/covered installments
├─ total_with_interest      -- final debt value
├─ total_paid               -- accumulated payments
├─ remaining_balance        -- unpaid balance
├─ status                   -- active | paused | capped | paid
└─ created_at               -- creation timestamp
```

### Example (Conceptual)

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
  status = "active"
}
```

---

## Hooks (Event Surface)

Hooks are the **primary reactive integration surface**.  
They are server-side, fire after authoritative state changes, and never affect execution flow.

### Available Hooks

```txt
LSHARK_ContractCreated(contractTable)
LSHARK_InstallmentPaid(code)
LSHARK_FineApplied(code)
LSHARK_LimitedContract(code)
LSHARK_ContractPaidOff(code)
LSHARK_ContractDeleted(code)
LSHARK_ContractStatusChanged(code, oldStatus, newStatus)
```

### Notes
- Hooks are emitted **after validation and persistence**.
- `contractTable` is provided only on creation; all others reference by `code`.
- Use hooks to sync restrictions, logs, webhooks, or external systems.

---

## Public Functions (`LSHARK.API`)

Functions provide **read-only queries** into the authoritative state.  
They never mutate contracts or balances.

### Contract Queries

```txt
GetAllContractsCode()            -> table<string>
GetAllContractData()             -> table<Contract>
GetContractsCode(steamid)        -> table<string>
GetContractData(code)            -> Contract | nil
```

### State & Balance Helpers

```txt
HasActiveDebt(steamid)           -> boolean
GetBalance(steamid)              -> number
```

### Notes
- `GetAllContractData()` is heavy and intended for admin usage only.
- `HasActiveDebt()` encodes the rule: remaining_balance > 0 and status != paid.
- `GetBalance()` returns the **LSHARK internal balance**, not DarkRP money.

---

## Integration Philosophy

- **Server-authoritative**
- **Event-driven via hooks**
- **Read-only data access via API**
- **No direct SQL or net usage required**

If you need to:
- **react** → use hooks  
- **query** → use `LSHARK.API`  
- **enforce rules** → combine hooks + queries  

This separation guarantees stability, forward compatibility, and safe third-party integration.
