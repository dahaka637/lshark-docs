# Public Hooks API (Server) — Contract Lifecycle Events

This document defines the **public hook interface** exposed by LSHARK for third-party integrations.  
Hooks are the recommended integration surface for addons that need to **react** to contract lifecycle changes without touching internal database, networking, or business logic.

All hooks are emitted server-side using:

- `hook.Run("<HookName>", ...)`

Consumers subscribe using:

- `hook.Add("<HookName>", "<UniqueListenerID>", function(...) ... end)`

---

## Design Rules (Integration Contract)

- Hooks are **server-only** signals. Client addons should not rely on them.
- Hook parameters are intentionally minimal and stable.
- Hooks are emitted **after** the authoritative action is validated and applied (and persisted when applicable).
- Hooks do not return values and do not affect execution flow (LSHARK does not read hook return values).

---

## Hook List

### 1) `LSHARK_ContractCreated`

**When it fires**  
After a debtor **accepts** a pending contract and the server:
- transfers funds
- persists the contract
- activates it (`status = "active"`)

**Signature**
```lua
hook.Run("LSHARK_ContractCreated", contractTable)
```

**Parameters**
- `contractTable` *(table)*: Full contract snapshot as stored by LSHARK at creation time.

**Contract table contains (typical keys)**
```txt
code, creditor, creditor_steamid, debtor, debtor_steamid, creditor_job,
amount, interest, fine, installments,
installments_paid, installments_advanced,
total_with_interest, total_paid, remaining_balance, status
```

**Notes**
- This is the main entry point for external systems (logging, restrictions, job tools).
- Fired only when the contract becomes authoritative (not when proposal is staged).

---

### 2) `LSHARK_InstallmentPaid`

**When it fires**  
Whenever a payment is successfully applied to a contract as an installment-related progression, including:
- automatic full installment payments (auto-collect)
- manual payments that progress installment state

**Signature**
```lua
hook.Run("LSHARK_InstallmentPaid", code)
```

**Parameters**
- `code` *(string)*: Contract code

**Notes**
- The hook signals that the contract’s payment progress moved forward.
- The payment amount is not included in this hook; consumers should query via API if needed.

---

### 3) `LSHARK_FineApplied`

**When it fires**  
When a fine is applied due to:
- automatic payment failure (no funds / no debit cap)
- partial payment where fine rules are applicable (not capped)

**Signature**
```lua
hook.Run("LSHARK_FineApplied", code)
```

**Parameters**
- `code` *(string)*: Contract code

**Notes**
- Not fired when the contract is already `capped` (fines are disabled in capped mode).
- Only indicates that a fine was applied; values must be queried if needed.

---

### 4) `LSHARK_LimitedContract`

**When it fires**  
When a contract becomes **capped/limited** due to reaching the multiplier ceiling. The server clamps remaining balance and disables fines.

**Signature**
```lua
hook.Run("LSHARK_LimitedContract", code)
```

**Parameters**
- `code` *(string)*: Contract code

**Notes**
- Fired on the transition into capped mode (not on every tick while capped).
- Indicates that contract escalation has been permanently bounded.

---

### 5) `LSHARK_ContractPaidOff`

**When it fires**  
When a contract reaches full settlement and becomes paid:
- remaining balance reaches 0
- status becomes `paid`

Occurs in:
- auto-collection final installment
- manual pay flow when final payment completes the debt

**Signature**
```lua
hook.Run("LSHARK_ContractPaidOff", code)
```

**Parameters**
- `code` *(string)*: Contract code

**Notes**
- Fired after persistence updates are applied.
- Useful for removing restrictions, releasing collateral, granting rewards, etc.

---

### 6) `LSHARK_ContractDeleted`

**When it fires**  
When a contract is deleted by:
- the creditor (owner delete)
- an admin action

Deletion includes:
- contract row removal
- associated logs cleanup
- associated analytics cleanup

**Signature**
```lua
hook.Run("LSHARK_ContractDeleted", code)
```

**Parameters**
- `code` *(string)*: Contract code

**Notes**
- Fired after the delete has been applied server-side.
- Consumers should treat this as final and remove any cached state.

---

### 7) `LSHARK_ContractStatusChanged`

**When it fires**  
When the creditor toggles a contract status and the change is persisted. Typical transitions include:
- `active ↔ paused`
- `capped ↔ paused` (for limited contracts)
- other policy-driven status flips (excluding `paid`, which is terminal)

**Signature**
```lua
hook.Run("LSHARK_ContractStatusChanged", code, oldStatus, newStatus)
```

**Parameters**
- `code` *(string)*: Contract code
- `oldStatus` *(string)*: Previous status (lowercased in toggle path)
- `newStatus` *(string)*: New persisted status

**Notes**
- Fired only after persistence succeeds.
- If the contract is limited, status toggling respects capped behavior rules.

---

## Typical Use Cases

- **Restriction systems**: lock doors, block job changes, or apply penalties while contracts are active.
- **Economy integrations**: tax, fee sharing, or alternative wallet bridges.
- **Discord / Webhook relays**: broadcast contract events in external systems.
- **Administrative tooling**: dashboards that react live to state changes.

---

## Minimal Example (Tester Pattern)

```lua
hook.Add("LSHARK_ContractCreated","MyAddon_OnCreated",function(contract)
    print("[MyAddon] contract created:", contract.code, contract.creditor_steamid, contract.debtor_steamid)
end)

hook.Add("LSHARK_ContractPaidOff","MyAddon_OnPaid",function(code)
    print("[MyAddon] contract paid off:", code)
end)

hook.Add("LSHARK_ContractStatusChanged","MyAddon_OnStatus",function(code, oldStatus, newStatus)
    print("[MyAddon] status changed:", code, oldStatus, "->", newStatus)
end)
```

---

## Compatibility Notes

- Hook names and parameter order must be treated as part of the public API.
- Integrations should avoid mutating the `contractTable` reference received in hooks.
- If an integration needs amounts or extra fields beyond the hook parameters, it should query the public server API (`LSHARK.API`) rather than reading internal DB tables directly.
