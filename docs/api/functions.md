# Public API — Server Functions (`LSHARK.API`)

This section documents the **public, server-side, read-only API** exposed by LSHARK for third-party addon integration. The goal is to allow external systems (inventories, restrictions, analytics, admin tools) to **observe contract state** without direct access to internal SQL, net messages, or business logic.

## Availability & Scope

- **Server-side only** (the module does not load on client).
- **Read-only access** to contract data (no create/update/delete functions).
- All contract payloads returned to addons are **sanitized copies**, so integrations don’t depend on internal DB schemas or row formats.

---

## Data Model Returned by the API

All functions that return contract data use the same **sanitized contract structure**.

```txt
Contract (sanitized)
├─ code (string)
├─ creditor (string)
├─ creditor_steamid (string)
├─ debtor (string)
├─ debtor_steamid (string)
├─ creditor_job (string)
├─ amount (number)                    -- principal
├─ interest (number)                  -- percent
├─ fine (number)                      -- percent
├─ installments (number)
├─ installments_paid (number)
├─ installments_advanced (number)
├─ total_with_interest (number)
├─ total_paid (number)
├─ remaining_balance (number)
├─ status (string)                    -- "active", "paused", "paid", "capped", "unknown", etc.
└─ created_at (any)                   -- forwarded from storage layer (if available)
```

Notes:
- Numeric fields are normalized (e.g., `tonumber(...) or 0/1`) so external code can treat them safely as numbers.
- `status` is always returned as a string (defaults to `"unknown"` if missing).

---

## Function Reference

### `LSHARK.API.GetAllContractsCode()`

Returns a list of **all contract codes** in the database. Intended for admin/audit addons.

**Signature**
```lua
local codes = LSHARK.API.GetAllContractsCode()
```

**Returns**
- `codes` (`table<string>`): zero or more contract codes.

**Notes**
- This is an administrative-scope call. Avoid frequent execution on busy servers.

---

### `LSHARK.API.GetAllContractData()`

Returns sanitized data for **all contracts**. This is a **heavy operation** and should be used only for admin panels or infrequent reporting.

**Signature**
```lua
local all = LSHARK.API.GetAllContractData()
```

**Returns**
- `all` (`table<Contract>`): array of sanitized contracts.

**Notes**
- Do not run per-tick or per-player. Prefer targeted calls (`GetContractsCode`, `GetContractData`).

---

### `LSHARK.API.GetContractsCode(steamid)`

Returns all contract codes related to a SteamID, including contracts where the player is **debtor or creditor**. Results are de-duplicated.

**Signature**
```lua
local codes = LSHARK.API.GetContractsCode("STEAM_0:1:123456")
```

**Parameters**
- `steamid` (`string`): SteamID to query. Empty/invalid returns `{}`.

**Returns**
- `codes` (`table<string>`): unique contract codes.

**Behavior**
- Collects codes from:
  - contracts where `steamid` is the **debtor**
  - contracts where `steamid` is the **creditor**
- Removes duplicates (same contract can match both sides in edge cases).

---

### `LSHARK.API.GetContractData(code)`

Returns sanitized contract data for a single contract code (or `nil` if not found).

**Signature**
```lua
local c = LSHARK.API.GetContractData("ABC-12345")
```

**Parameters**
- `code` (`string`): contract code. Empty/invalid returns `nil`.

**Returns**
- `Contract|nil`: sanitized contract table, or `nil` if it doesn’t exist.

---

### `LSHARK.API.GetBalance(steamid)`

Returns the **internal LSHARK balance** of a player.  
This is the balance used by LSHARK for lending operations and **is not** the DarkRP wallet balance.

**Signature**
```lua
local balance = LSHARK.API.GetBalance("STEAM_0:1:123456")
```

**Parameters**
- `steamid` (`string`): Player SteamID. Empty or invalid values return `0`.

**Returns**
- `number`: Current LSHARK internal balance (always ≥ 0).

**Notes**
- This value represents the lender’s internal funds stored by LSHARK.
- It is used when issuing contracts and when receiving installment payments.
- External addons must use this function instead of accessing database helpers directly.

---

### `LSHARK.API.HasActiveDebt(steamid)`

Returns `true` if the SteamID has **any debt** as debtor, defined by:
- `remaining_balance > 0`, and
- `status ~= "paid"`

**Signature**
```lua
local hasDebt = LSHARK.API.HasActiveDebt("STEAM_0:1:123456")
```

**Parameters**
- `steamid` (`string`): debtor SteamID. Empty/invalid returns `false`.

**Returns**
- `boolean`: whether any non-paid contract still has remaining balance.

**Notes**
- This is the recommended helper for restrictions (job changes, purchases, rentals) when you only need a yes/no answer.

---

## Usage Examples (Server-Side)

### Example 1 — Block an action if player has debt
```lua
hook.Add("PlayerCanBuyPistol", "LSHARK_BlockIfDebt", function(ply)
    if not IsValid(ply) then return end
    if LSHARK.API.HasActiveDebt(ply:SteamID()) then return false, "You cannot buy this while in debt." end
end)
```

### Example 2 — Fetch all contracts for a player and print remaining balance
```lua
local function DebugPrintPlayerContracts(ply)
    if not IsValid(ply) then return end
    local codes = LSHARK.API.GetContractsCode(ply:SteamID())
    for _, code in ipairs(codes) do
        local c = LSHARK.API.GetContractData(code)
        if c then print("[LSHARK API]", c.code, c.status, c.remaining_balance) end
    end
end
```

### Example 3 — Admin scan (heavy): list all contract codes
```lua
concommand.Add("lshark_admin_list_codes", function(ply)
    if IsValid(ply) and not ply:IsAdmin() then return end
    local codes = LSHARK.API.GetAllContractsCode()
    print("[LSHARK API] Total contracts:", #codes)
end)
```

---

## Best Practices for Integrators

- Prefer **targeted queries**: `GetContractsCode(steamid)` + `GetContractData(code)`.
- Use `GetAllContractData()` only for **admin-only** panels or periodic exports.
- For restrictions, prefer `HasActiveDebt()` over manual status checks, because it encodes the rule “remaining > 0 and not paid”.
- Avoid caching contract tables for long periods; prefer refreshing on LSHARK hooks (e.g., `LSHARK_ContractCreated`, `LSHARK_ContractPaidOff`, `LSHARK_ContractStatusChanged`).
