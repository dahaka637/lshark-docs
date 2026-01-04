## LSHARK – Architecture Overview

LSHARK is a modular loan and debt management system designed for Garry’s Mod (DarkRP), built around a strict separation of responsibilities between core logic, data storage, networking, client interfaces, and public integrations. The system centralizes all critical operations on the server side, exposes a controlled and read-only public API for third-party addons, and relies on event-driven hooks to notify external systems about contract lifecycle changes. This architecture ensures stability, security, and extensibility, allowing other addons to integrate with LSHARK without direct access to internal database structures, networking layers, or business logic.

<pre>
📁 lshark/
├── 📁 lua/
│   ├── 📁 autorun/
│   │   └── 📄 lshark_loader.lua
│   ├── 📁 entities/
│   │   └── 📁 ent_auto_loan/
│   │       ├── 📄 cl_init.lua
│   │       ├── 📄 init.lua
│   │       └── 📄 shared.lua
│   ├── 📁 lshark/
│   │   ├── 📁 api/
│   │   │   └── 📄 sv_api.lua
│   │   ├── 📁 client/
│   │   │   ├── 📁 tabs/
│   │   │   │   ├── 📄 cl_tab_analytics.lua
│   │   │   │   ├── 📄 cl_tab_contracts.lua
│   │   │   │   ├── 📄 cl_tab_logs.lua
│   │   │   │   └── 📄 cl_tab_management.lua
│   │   │   ├── 📄 cl_admin.lua
│   │   │   ├── 📄 cl_auto_loan.lua
│   │   │   ├── 📄 cl_boot.lua
│   │   │   ├── 📄 cl_client.lua
│   │   │   ├── 📄 cl_gui_framework.lua
│   │   │   └── 📄 cl_panel.lua
│   │   ├── 📁 communication/
│   │   │   └── 📄 sv_notifications.lua
│   │   ├── 📁 contracts/
│   │   │   ├── 📄 sv_contract_autocollect.lua
│   │   │   ├── 📄 sv_contract_manager.lua
│   │   │   └── 📄 sv_contract_storage.lua
│   │   ├── 📁 core/
│   │   │   ├── 📄 sv_analytics.lua
│   │   │   ├── 📄 sv_database.lua
│   │   │   ├── 📄 sv_logs.lua
│   │   │   └── 📄 sv_network.lua
│   │   ├── 📁 finance/
│   │   │   ├── 📄 sv_auto_loan.lua
│   │   │   ├── 📄 sv_lender_balance.lua
│   │   │   └── 📄 sv_manual_payments.lua
│   │   ├── 📁 localization/
│   │   │   ├── 📄 lang_en.lua
│   │   │   ├── 📄 lang_br.lua
│   │   │   ├── 📄 lang_es.lua
│   │   │   ├── 📄 lang_fr.lua
│   │   │   ├── 📄 lang_de.lua
│   │   │   ├── 📄 lang_it.lua
│   │   │   ├── 📄 lang_pl.lua
│   │   │   ├── 📄 lang_ru.lua
│   │   │   ├── 📄 lang_tr.lua
│   │   │   ├── 📄 language_manager.lua
│   │   │   └── 📄 sv_language_manager.lua
│   │   └── 📄 settings.lua
│   └── 📁 weapons/
│       └── 📄 lshark_clipboard.lua
├── 📁 materials/
│   ├── 📁 entities/
│   │   └── 🖼️ auto_loan.png
│   ├── 📁 gui/
│   │   ├── 🖼️ contract.png
│   │   ├── 🖼️ error.png
│   │   ├── 🖼️ info.png
│   │   ├── 🖼️ logo.png
│   │   ├── 📄 logo.vmt
│   │   ├── 📄 logo.vtf
│   │   ├── 🖼️ money.png
│   │   ├── 🖼️ success.png
│   │   ├── 🖼️ tips.png
│   │   └── 🖼️ warning.png
│   ├── 📁 models/
│   │   └── 📁 jason278/
│   │       └── 📁 postal 2/
│   │           └── 📁 weapons/
│   │               ├── 📄 clipboard_timb.vmt
│   │               └── 📄 clipboard_timb.vtf
│   └── 📁 vgui/
│       └── 📁 entities/
│           ├── 🖼️ lshark_autoloan.png
│           └── 🖼️ lshark_clipboard.png
├── 📁 models/
│   └── 📁 postal 2/
│       └── 📁 weapons/
│           ├── 📄 clipboard.dx80.vtx
│           ├── 📄 clipboard.dx90.vtx
│           ├── 📄 clipboard.mdl
│           ├── 📄 clipboard.phy
│           ├── 📄 clipboard.sw.vtx
│           └── 📄 clipboard.vvd
└── 📁 sound/
    └── 📁 lshark/
        └── 📁 contract/
            └── 🎵 paper_sign.wav
</pre>

### lshark_loader.lua — Core Bootstrap & Module Loader

The `lshark_loader.lua` file is the central bootstrap of the entire LSHARK system and is responsible for initializing the global namespaces, validating the runtime environment, and orchestrating the loading order of all server-side and client-side modules. It enforces DarkRP compatibility at startup, initializes shared configuration (`settings.lua`), registers workshop dependencies, and defines helper utilities for controlled file loading with optional debug output. This loader guarantees a strict separation of concerns by loading core systems (database, logs, analytics, networking) before higher-level finance, contract, API, and communication layers, while also explicitly declaring which files are sent to clients. Its role is critical: without it, no other subsystem, API, hook, or UI component of LSHARK can function or be safely exposed.


## Loan Contract Entity (`ent_auto_loan`)

The **Loan Contract Entity** represents a physical, in-world manifestation of a loan contract inside Garry’s Mod.  
It acts as an interaction anchor between players and the LSHARK loan system, visually and logically linking gameplay actions to the underlying contract mechanics.

---

### `shared.lua`

Defines the **core metadata and identity** of the entity, shared between server and client.  
This file declares the entity type, base class, spawn behavior, category, and spawn menu icon.  
It ensures the entity is correctly registered in Garry’s Mod and recognized as a spawnable LSHARK component, without containing any logic or rendering code.

**Responsibilities:**
- Entity registration  
- Spawn menu configuration  
- Shared identity and classification  

---

### `init.lua` (Server-side)

Handles the **server-side lifecycle and ownership logic** of the entity.  
This file is responsible for initializing the entity in the world, assigning ownership, and ensuring it integrates correctly with DarkRP and the LSHARK backend systems.

It acts as the server authority layer, guaranteeing that the entity exists as a valid contract-related object and can safely interact with server logic without exposing internal systems to the client.

**Responsibilities:**
- Server-side initialization  
- Ownership handling  
- Integration with gameplay rules and contract logic  

---

### `cl_init.lua` (Client-side)

Implements the **visual representation** of the contract entity using a 3D2D interface.  
This file renders floating information above the entity, including the contract title, icon, and creditor name, while applying distance checks and optimized rendering to avoid unnecessary performance costs.

No business logic or contract data manipulation occurs here; it is strictly presentation-focused.

**Responsibilities:**
- Client-side rendering  
- 3D2D visual interface  
- Performance-safe visual feedback  

## Public Server API (`sv_api.lua`)

This file exposes the **official public server-side API** of LSHARK, designed for safe third-party integration.  
It provides **read-only access** to loan contract data while fully isolating internal systems such as database access, networking, and business logic.

All functions are registered under the `LSHARK.API` namespace and are available **server-side only**.  
Contract data is always returned in a **sanitized and stable format**, ensuring external addons do not depend on internal schemas.

The API is intended for administrative tools, analytics, restrictions, and external systems that need to **observe** the state of contracts without mutating them.


## Client Interface Layer (UI)

The **client layer** of LSHARK is responsible for all user-facing interfaces, visual rendering, and interaction flow.  
It operates strictly as a **presentation and interaction layer**, containing no business rules, persistence logic, or authoritative state changes.  
All actions are executed through validated server communication.

---

### `cl_boot.lua` — Client Bootstrap & UI Entry Point

Initializes the client-side UI environment.  
Defines shared helper utilities (currency and percentage parsing/formatting), registers the main menu command (`lshark_menu`), enforces job and equipment requirements, and mounts the core interface tabs.  
Also handles generic server notifications and requests the initial lender balance after client initialization.

---

### `cl_panel.lua` — Debtor Interface

Implements the **debtor-facing panel** used to view and manage personal debts.  
Renders a debt table, detailed contract inspection panel, and manual payment controls.  
All data is provided by the server, and all actions (payments, limits) are sent back via validated net messages.  
Contains no validation or financial logic; strictly UI and user input handling.

---

### `cl_client.lua` — Contract Reception & Signing Flow

Handles the **client-side contract interaction lifecycle**.  
Receives contract offers from the server, builds presentation data, opens the contract signing interface, and captures accept/reject actions.  
Supports keyboard shortcuts and automatic opening for auto-loan contracts.  
All decisions and state changes are delegated to the server.

---

### `cl_auto_loan.lua` — Auto Loan Interface

Implements the **automatic loan creation UI**.  
Allows players to simulate loan values, installments, interest, penalties, and available credit using server-supplied configuration limits.  
Maintains local caches for configuration and existing debt strictly for visual previews.  
All calculations are non-authoritative; final validation and contract creation occur server-side.

---

### `cl_admin.lua` — Administrative Interface

Provides the **administrative contract management UI**.  
Displays all contracts, supports filtering and inspection, and allows deletion through confirmed server requests.  
Acts purely as an administrative presentation layer with no local authority over contract state.

---

### `cl_gui_framework.lua` — GUI Framework Core

Defines the **shared UI framework** used across all client interfaces.  
Provides dynamic scaling, centralized font management, a global color palette, and reusable UI primitives such as frames, tables, buttons, popups, and specialized components.  
Includes custom text rendering utilities for advanced layouts.  
Contains no business logic; responsible only for layout, styling, and UI composition.

---

### `cl_tab_management.lua` — Loan Management Tab

Implements the **lender management interface**.  
Allows configuration of auto-loan parameters, management of lender balance (deposit/withdraw), and creation of manual loan offers.  
All previews are client-side only; limits, cooldowns, and contract creation are enforced server-side.

---

### `cl_tab_contracts.lua` — Contracts Overview Tab

Displays a **consolidated view of all contracts** with status-aware visuals.  
Provides detailed inspection of selected contracts and supports lifecycle actions (toggle, refresh, delete) through validated server messages.  
Serves as the primary contract inspection UI without local authority.

---

### `cl_tab_logs.lua` — Activity Logs Tab

Implements the **audit and activity log viewer**.  
Receives serialized log data from the server, provides real-time search and sorting, and supports manual refresh.  
All logs are generated and classified server-side.

---

### `cl_tab_analytics.lua` — Analytics & Statistics Dashboard

Implements the **financial analytics dashboard** using an embedded HTML (Chart.js) interface.  
Displays KPIs, time-series charts, profit/loss indicators, and interactive time-range selection.  
Processes server-provided analytics data strictly for visualization, with no local financial authority.

---

**Summary:**  
The client layer of LSHARK is intentionally thin, modular, and non-authoritative.  
Its sole responsibility is to translate server-controlled state into clear, responsive, and localized user interfaces, while delegating all validation, persistence, and security decisions to the server.

## communication

### sv_notifications.lua — Server Notifications & Player Feedback

This file implements the **server-side notification and feedback layer** of LSHARK, responsible for sending contextual messages and guidance to players based on server-side events and interactions. It centralizes non-intrusive communication logic without exposing business rules or internal systems.

The module defines a controlled helper for dispatching notifications to clients via the `LSHARK_Notification` network channel, allowing the server to trigger informational, warning, or tip messages that are rendered client-side through the UI framework.

It also registers lightweight **chat command shortcuts**, enabling players to open the debtor panel by typing `lshark` (or common typo variations) directly in chat. This improves accessibility while remaining fully server-controlled.

Additionally, the file listens for **job change events** (`OnPlayerChangedTeam`) and checks them against configured lender job rules. When a player becomes eligible as a lender, a contextual tip notification is sent, informing them of newly available LSHARK features.

Overall, this file acts as the **server-driven user guidance and notification bridge**, enhancing usability and discoverability while maintaining strict separation from contract logic, persistence, and validation systems.

## contracts

### sv_contract_storage.lua — Contract Persistence & Database Access Layer

This file implements the **server-side persistence layer** for LSHARK loan contracts.  
It is responsible exclusively for **storing, retrieving, updating, and deleting contract data** in the local SQLite database, acting as the single source of truth for contract state persistence.

The module provides a set of well-defined database helper functions under the `LSHARK.DB_*` namespace, encapsulating all SQL operations and shielding the rest of the system from raw database queries. Input values are sanitized and normalized before execution, and all write operations are executed through a protected wrapper to prevent server crashes caused by SQL errors.

Core responsibilities include:
- Inserting and updating full contract records.
- Fetching contracts by code, creditor, debtor, or status.
- Persisting payment progress, remaining balance, and contract status.
- Supporting administrative operations such as full listings, deletion, and global cleanup.
- Toggling contract state (active ↔ paused) in a controlled manner.

This file contains **no business rules, validation logic, or gameplay decisions**.  
It serves strictly as the **data storage backend**, enabling higher-level systems (contract manager, auto-collection, analytics, API) to operate on reliable, persistent contract data without direct database coupling.

### sv_contract_manager.lua — Contract Validation, Lifecycle & Authority Layer

This file implements the **core server-side contract management layer** of LSHARK.  
It acts as the **authoritative controller** for the entire contract lifecycle, handling validation, creation, acceptance, rejection, status changes, deletion, and administrative actions. All contract-related decisions pass through this module.

The module validates **job permissions, financial limits, interest and fine constraints, installment rules, and available balance** before allowing any contract to be issued. Both manual and auto-loan contracts are supported, with auto-loans enforcing additional configuration-based restrictions.

Contracts are first stored in a **pending state** and only become persistent after explicit debtor acceptance. Upon acceptance, the module performs balance transfers, persists the contract through the storage layer, registers logs and analytics events, and triggers system-wide hooks for external integrations.

The file also manages:
- Contract acceptance and rejection flow.
- Controlled toggling of contract status (active, paused, capped, paid).
- Creditor-owned contract deletion.
- Full administrative contract inspection and deletion.
- Server notifications for all contract outcomes.
- Emission of lifecycle hooks (`LSHARK_ContractCreated`, `LSHARK_ContractStatusChanged`, `LSHARK_ContractDeleted`) for extensibility.

No client input is trusted directly.  
This file is the **central authority enforcing rules, integrity, and consistency** across all contract operations in LSHARK, coordinating storage, finance, analytics, logging, notifications, and public hooks.

### sv_contract_autocollect.lua — Automatic Debt Processing & Enforcement Engine

This file implements the **automatic debt collection system** of LSHARK and is one of the most critical server-side enforcement components.  
It is responsible for periodically processing debtor contracts, applying automatic payments, partial payments, penalties, debt caps, and final contract resolution in a fully server-authoritative manner.

The module evaluates each active or capped contract for online players at fixed intervals, calculating how much can be safely debited based on the debtor’s available cash and configured auto-debit limits. When possible, installments are automatically paid and credited to the lender’s balance; otherwise, fines or partial payments are applied according to contract rules.

Key responsibilities include:
- Enforcing debtor auto-debit limits and percentage caps.
- Applying installment payments, partial payments, or fines when payments fail.
- Capping debt growth based on job-defined loan multipliers.
- Marking contracts as paid, capped, or limited when thresholds are reached.
- Registering logs and analytics events for every automated action.
- Emitting lifecycle hooks (`LSHARK_InstallmentPaid`, `LSHARK_FineApplied`, `LSHARK_ContractPaidOff`, `LSHARK_LimitedContract`) for external integrations.

The system runs on a configurable server timer and restarts automatically on map load or cleanup. A debug-only command is provided to force immediate processing during development or testing.

This file acts as the **automated enforcement engine** of LSHARK, ensuring debts evolve consistently, fairly, and predictably over time without client involvement, while maintaining full auditability and extensibility.

## core

### sv_database.lua — Core Database Initialization & Low-Level Access

This file defines the **database foundation** of LSHARK.  
It is responsible for initializing all required SQLite tables, creating indexes, and exposing a minimal set of low-level database helpers used by higher-level storage and core systems. 

On server startup, the module ensures that every persistence table required by LSHARK exists, including contracts, balances, logs, analytics events, auto-loan configuration, debtor limits, and limited-contract tracking. Indexes are created for frequently queried fields to improve lookup performance and scalability.

The file also exposes controlled helper functions for executing queries, fetching result sets, and checking record existence. These helpers act as a **thin abstraction layer** over Garry’s Mod’s SQLite API, allowing other modules to interact with the database without duplicating boilerplate or unsafe SQL handling.

No business logic, validation rules, or gameplay decisions are present in this file.  
It serves strictly as the **core persistence bootstrap and database utility layer**, upon which all other server-side storage, analytics, logging, and contract systems depend.

### sv_analytics.lua — Financial Analytics & Event Tracking Core

This file implements the **server-side analytics system** of LSHARK.  
It is responsible for recording, storing, maintaining, and serving financial and contract-related events that feed the client analytics dashboard and administrative insights.

The module exposes a controlled API to **register analytics events** tied to a player and contract, capturing loaned amounts, received payments, open balances, and event timestamps. All events are persisted in the analytics table using normalized Unix timestamps to ensure consistent chronological processing.

To maintain performance and data relevance, the file includes automated **cleanup routines** that limit the number of tracked contracts per player and remove orphaned analytics entries when contracts no longer exist. These routines run periodically and on server initialization without player involvement.

The module also handles **client analytics requests**, serializing and sending the complete event timeline for the requesting player in JSON format. No aggregation or visualization logic exists here; all higher-level interpretation is performed client-side.

This file functions as the **analytics data backbone** of LSHARK, providing a clean, bounded, and reliable event history that supports dashboards, audits, and external integrations without affecting core gameplay logic.


### sv_logs.lua — Audit Logs & Event History Core

This file implements the **server-side logging system** of LSHARK.  
It is responsible for recording, storing, maintaining, and serving **audit logs** related to contracts, payments, penalties, and administrative actions, providing a reliable historical trail for users and administrators.

The module exposes a controlled function (`LSHARK.RegisterLog`) used by other server systems to persist log entries associated with a player and, optionally, a specific contract. Each log entry records the event type, human-readable message, and timestamp, ensuring traceability of all relevant actions.

It also handles **client log requests**, returning a capped, ordered list of recent log entries for the requesting player. Logs are sent in a pre-serialized format optimized for client-side rendering, without exposing database internals.

To preserve performance and storage health, the file includes automated **maintenance routines**:
- Periodic cleanup to limit the maximum number of logs stored per player.
- Removal of orphaned log entries when related contracts no longer exist.
- Automatic pruning of invalid or empty contract references.

This file contains no gameplay logic or validation rules.  
It functions strictly as the **audit and traceability backbone** of LSHARK, ensuring transparency, accountability, and historical insight across all financial and contractual operations.

### sv_network.lua — Network Channel Registration & Communication Boundary

This file defines the **network communication boundary** of LSHARK.  
It centralizes the registration of all network strings used for client–server communication, ensuring that every message channel is explicitly declared, organized, and auditable. 

The module groups network channels by functional domain, including user interface actions, contract lifecycle operations, administrative controls, analytics and logs, auto-loan systems, balance management, and debtor interactions. This structure enforces clarity and prevents ad-hoc or duplicated network definitions across the codebase.

No message handling, validation, or logic is implemented here.  
The file exists solely to **declare and expose the allowed communication channels**, serving as a strict interface contract between the client and server layers.

By centralizing all `util.AddNetworkString` calls, this file strengthens maintainability, security, and code reviewability, making it clear which interactions are permitted and where they are handled.

## finance

### sv_manual_payments.lua — Manual Debt Payments & Debtor Controls

This file implements the **server-side manual payment system** of LSHARK.  
It handles all debtor-initiated payments, allowing players to manually settle debts by amount or by installments while enforcing full server authority over validation, balance updates, and contract state changes. 

The module validates debtor identity, contract ownership, available funds, remaining balance, and payment method before applying any transaction. It supports both **full and partial payments**, correctly updating installments paid, advanced installments, remaining balance, and final contract status when fully settled.

In addition, the file manages **debtor auto-debit limits**, allowing players to configure how much of their balance can be used for automatic collections. These limits are persisted server-side and respected by the automatic debt processing engine.

Every manual payment triggers:
- Contract state updates in persistent storage.
- Secure money transfer between debtor and creditor balance.
- Log registration for auditability.
- Analytics event registration for financial tracking.
- User notifications for both debtor and creditor.

This file contains no client-side logic and does not trust client input.  
It serves as the **authoritative manual settlement and debtor control layer**, complementing the automatic collection system while maintaining consistency, traceability, and financial integrity.

### sv_lender_balance.lua — Lender Balance Management & Internal Wallet

This file implements the **server-side lender balance system** of LSHARK.  
It manages the internal wallet used by creditors to fund loans, receive repayments, and perform balance transfers independently from the player’s DarkRP cash. 

The module defines a persistent balance per SteamID and exposes controlled helper functions to query, add, set, and deduct balance values while preventing invalid or negative states. Balance rows are created lazily to ensure data consistency without manual initialization.

It also handles **client-initiated balance operations**, allowing lenders to deposit DarkRP money into their LSHARK balance or withdraw funds back to their wallet. All transactions are validated server-side, enforcing sufficient funds and valid amounts before any transfer occurs.

After each operation, the updated balance is sent back to the client, keeping the UI synchronized with the authoritative server state. Notifications are emitted to inform the player of successful or failed actions.

This file contains no contract logic or gameplay rules.  
It functions strictly as the **financial wallet and balance authority layer** for lenders, forming the monetary backbone that supports contract creation, repayments, and analytics across the LSHARK system.

### sv_auto_loan.lua — Automatic Loan Configuration & Issuance Core

This file implements the **server-side automatic loan system** of LSHARK.  
It is responsible for storing auto-loan configurations, validating auto-loan requests, calculating limits, and issuing auto-generated contracts under strict server authority. 

The module persists per-creditor auto-loan settings, including maximum loan amount, maximum installments, interest rate, and penalty rate. These configurations are securely stored and retrieved from the database and exposed to clients only in a sanitized, read-only form for interface previews.

It provides helper functions to:
- Load and normalize auto-loan configuration data.
- Calculate active debt between a specific debtor and creditor.
- Enforce credit limits and installment constraints before issuing a loan.

The file handles all **auto-loan network requests**, including configuration retrieval, debt queries, and loan submission. Before creating any contract, it validates that the requested amount does not exceed configured limits or existing debt capacity.

When an auto-loan request is approved, the module delegates contract creation to the central contract manager, generating standardized auto-loan contracts with consistent metadata and lifecycle handling.

This file contains no client-side logic and does not trust client input.  
It serves as the **configuration, validation, and issuance layer** for automatic loans, tightly integrated with balance management, contract enforcement, analytics, and logging systems.

## Localization System (I18N)

The localization layer of **LSHARK** provides a unified and extensible internationalization system, ensuring that all user-facing text is language-aware, consistent, and configurable at runtime.  
It is designed to work transparently across client and server, without embedding hardcoded strings in business or UI logic.

---

### `language_manager.lua` — Client Language Manager

Implements the **client-side localization controller**.  
Responsible for registering language packs, selecting the active language, resolving translation keys, and formatting localized strings with parameters.

The manager supports graceful fallback to English when a key is missing and automatically applies the language defined in `settings.lua` during client initialization.  
All UI components access translations exclusively through this layer.

---

### `sv_language_manager.lua` — Server Language Manager

Implements the **server-side localization controller**, mirroring the client behavior for server-originated messages such as notifications, logs, and chat output.

The active language is loaded from server configuration and applied globally, ensuring consistency between server messages and client UI text.

---

### Language Packs

Each language file registers a dictionary of translation keys mapped to localized strings.  
They contain **no logic**, only structured key–value pairs.

Supported languages:

- `lang_en.lua` — English (default / fallback)
- `lang_br.lua` — Portuguese (Brazil)
- `lang_es.lua` — Spanish
- `lang_fr.lua` — French
- `lang_de.lua` — German
- `lang_it.lua` — Italian
- `lang_pl.lua` — Polish
- `lang_ru.lua` — Russian
- `lang_tr.lua` — Turkish

---

**Summary:**  
The localization system centralizes all translatable text, enforces consistent language resolution across client and server, and allows LSHARK to be deployed in multilingual environments without modifying core logic or interfaces.


## settings.lua — Global Configuration & Rule Definition

This file defines the **central configuration layer** of LSHARK.  
It is the single source of truth for system behavior, limits, intervals, language selection, and job-specific financial rules. All server-side validation logic ultimately derives its constraints from this file. 

---

### General Settings

The `General` section controls **global system behavior** and default safeguards:

- **DebugMode**: Enables development-only features and verbose output. Must remain disabled in production.
- **Language**: Defines the default language used by both client and server localization systems.
- **DefaultDebtDiscount**: Percentage of a debtor’s balance eligible for automatic debt collection.
- **AutoDebtProcessingInterval**: Time interval (in seconds) for automatic debt processing cycles.

It also defines **global fallback limits**, applied when a job has no specific override:
- Maximum total interest
- Maximum interest per installment
- Maximum late fee percentage
- Maximum number of installments
- Maximum debt multiplier over the original loan value

These values act as baseline safety limits to prevent abusive or unstable configurations.

---

### JobLimits — Profession-Based Rules

The `JobLimits` section defines **job-specific financial rules** that override global defaults.  
Each job entry controls how that profession may issue loans, including:

- **MaxInterest**: Base guaranteed cap for total interest.
- **MaxInterestPerInstallment**: Per-installment interest ceiling.
- **MaxFine**: Maximum late payment penalty percentage.
- **MaxLoanInstallments**: Upper bound for installment count.
- **MaxLoanMultiplier**: Absolute cap on total debt growth.

The effective maximum interest is dynamically calculated as the **greater** of:
- `MaxInterest`
- `MaxInterestPerInstallment × NumberOfInstallments`

This model allows flexible scaling for long-term contracts while still protecting short-term loans from extreme rates.

If accumulated interest or fines exceed the multiplier cap, the contract is automatically **capped**, and further penalties are disabled to prevent infinite debt growth.

---

### Role in the Architecture

This file contains **no logic** and performs **no runtime actions**.  
Its responsibility is strictly declarative.

All validation, enforcement, auto-collection, contract creation, and UI previews rely on these values, making `settings.lua` the **policy backbone** of the entire LSHARK system.

Changing this file alters system behavior globally without requiring code changes elsewhere.

## Clipboard SWEP

### lshark_clipboard.lua — LSHARK Clipboard SWEP (UI Access Tool)

This file implements the **LSHARK clipboard SWEP**, which serves as the **physical interaction tool** that grants players access to the LSHARK management interface. It bridges in-world gameplay interaction with the client UI layer, without embedding any business or financial logic. 

The weapon is configured as a utility SWEP, using a clipboard world model and custom iconography to visually represent contract handling. It disables combat-related features such as ammo, crosshair, and damage, reinforcing its purely administrative purpose.

On the client side, the SWEP handles **custom viewmodel and worldmodel rendering**, replacing the default weapon model with a properly aligned clipboard using bone-based positioning and transformation matrices. This ensures consistent visual presentation in both first-person and third-person views.

The primary action (`PrimaryAttack`) is server-authoritative and simply triggers the `lshark_menu` console command, opening the main LSHARK interface if the player meets all server-side permission checks. A small cooldown prevents interaction spam, and an audio cue provides immediate feedback.

No contract logic, validation, or state mutation exists in this file.  
Its responsibility is strictly **UI access and visual representation**, acting as the final interaction layer that connects players to the LSHARK system in a natural, immersive way.
