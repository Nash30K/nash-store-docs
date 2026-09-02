# Changelog

All notable changes to Nash Banking are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/) and the project adheres to [Semantic Versioning](https://semver.org/).

---

## [1.1.0] - External business system integration (TPE deposit hooks)

### Added

- **`Config.TPE.BusinessDepositHook`** and **`Config.TPE.EmployeeDepositHook`** - two optional server-side hooks in `shared/config.business.lua` to route TPE payment receipts to an external business system (`esx_society`, `qb-management`, custom scripts, ...) instead of Nash Banking's internal `nash_businesses.balance` or the seller's personal bank account. `BusinessDepositHook` fires when the TPE was purchased through Nash Banking's business panel; `EmployeeDepositHook` fires for generic TPE items and receives the seller's `job.name` so you can dispatch by job. Both default to `nil`, in which case the existing behavior is preserved 100%. Returning `true` from a hook tells Nash Banking your code handled the credit, and both the internal balance credit AND the internal transaction log are skipped for that payment. Notifications and Discord logs are still emitted. See the updated [config.business.lua](config/config-business.md) reference for the full signatures and copy-paste examples.

---

## [1.0.8] - Configurable RIB, ATM action toggles, phone auto-refresh after registration

### Added

- **`Config.BankInfo`** - the **Bank details** modal (IBAN / BIC / bank name and address / correspondent BIC) is now fully configurable through a dedicated `shared/config.bankinfo.lua` file. Set `IbanCountry` to `US`, `GB`, `DE`, `CH`, etc. and adapt the surrounding fields to match your server's country. See the new [config.bankinfo.lua](config/config-bankinfo.md) reference page.
- **`Config.ATM.AllowDeposit`** and **`Config.ATM.AllowWithdraw`** - two new options in `shared/config.atm.lua` to disable the deposit and / or withdraw buttons on ATMs independently. Hidden button-side and refused server-side, so no client bypass. Common use: force players to come to the bank for deposits by setting `AllowDeposit = false`. See the updated [config.atm.lua](config/config-atm.md) reference.

### Fixed

- **Phone banking app stayed on "Account required" after opening a bank account** - new players who registered at the bank NPC used to see the "Account required" screen on the phone banking app until they restarted the script or reconnected. The server now broadcasts `nash_banking:playerRegistered` on successful registration; `nash_banking_phone` forwards it to the NUI as `accountRegistered` and the React context refetches its state immediately.

### Changed

- **`BankInfoModal` no longer reads bank RIB fields from `useLocale.ts`** - the four locale keys `bank_iban_value`, `bank_bic_value`, `bank_address`, `bank_correspondent_bic_value` were removed from `fr` and `en` blocks. The modal now reads exclusively from `Config.BankInfo` sent through the `openBank` NUI payload. This gives one single source of truth for these values (the Lua config file), regardless of the active language.

---

## [1.0.7] — Quasar Phone V3, character switch, runtime locales

### Added

- **Quasar Phone V3 support** — `nash_banking_phone` and `nash_businessbanking_phone` now auto-detect the current `qs-smartphone` resource (Quasar Phone V3) in addition to LB Phone and legacy `qs-smartphone-pro`. Detection order: `lb-phone` → `qs-smartphone` → `qs-smartphone-pro`. Uses the current V3 `addCustomApp` signature (`id` / `label` / `icon` / `iframe.url` / `appStoreOnly` / `price`). Prints the detected system in F8 on boot (`[Nash Banking Phone] Phone system: quasar`).
- **Runtime locales for the desktop UI** — the desktop banking React NUI now merges runtime locale overrides on top of its compiled `fr` / `en` defaults. Adding a new language (e.g. Slovenian, Spanish, German) no longer requires the React source or a rebuild — just drop a `locales/<lang>.lua` file, set `Config.Locale`, restart. See the updated [Customize locales](guides/customize-locales.md) guide.
- **`GetLocaleTable()` global helper** in `shared/locale.lua` — returns the active `Locales[Config.Locale]` table as a Lua map. Used internally to feed the NUI payload; also available for third-party integrations that want the same fallback logic.
- **`Bridge.GetAllJobs()` and `Bridge.JobExists()` framework-agnostic helpers** — read jobs list from ESX `jobs` table or from `FrameworkObj.Shared.Jobs` on QBCore / QBox. Used internally by the admin panel to power the "create job-linked business" flow across all frameworks.

### Fixed

- **Phone apps returned to a blank screen after a multichar switch** — when a player switched character without disconnecting, the phone iframe kept the previous character's React state and the new (unregistered) character saw the background but no UI content. Fix: `nash_banking_phone/client.lua` and `nash_businessbanking_phone/client.lua` now listen for `esx:playerLoaded` / `QBCore:Client:OnPlayerLoaded` and fire a `characterSwitch` NUI event that resets the React state and re-fetches for the new character.
- **`isRegistered` default in the personal phone context was `true`** — caused a brief flash of the empty main dashboard for unregistered new characters. Default is now `false` so unregistered players immediately see the "Register at a bank" screen.
- **Job-linked business creation crashed on QBCore / QBox** — the admin panel's "create business for a job" flow used to query the ESX-legacy `jobs` and `addon_account_data` tables directly, which don't exist on QB / QBox. All 5 callsites in `server/database.lua` and 1 in `server/main.lua` now gate the ESX SQL path by `Framework == 'esx'` and fall back to reading / writing balance in `nash_businesses.balance` (same source of truth as personal businesses). Existing ESX behavior is unchanged.
- **Bank actions ("Insufficient funds" shown for every failure)** — `deposit`, `withdraw`, `transfer`, `moveSavings`, and `moveToBank` server callbacks now return a specific error code (`err_amount_too_high`, `err_daily_limit_reached`, `err_rate_limited`, `insufficient_cash`, `insufficient_funds`, `insufficient_savings`, `invalid_amount`) instead of a generic boolean. The desktop and phone Action modals now show the exact reason instead of a misleading "Insufficient funds" for cases like subscription limits or rate limits.
- **Notification toast rendered as an opaque black block on some setups** — the CSS `backdrop-filter: blur(20px)` isn't supported by every FiveM CEF build / GPU driver combo. Added a solid semi-opaque dark fallback under the gradient so the notification remains readable regardless of whether blur renders. Blur still applies for the premium look when supported.
- **`daily_limit_exceeded` TPE notification message key mismatch** — the fail toast used the "approaching the limit" copy (`notif_limit_warning_desc`) instead of a proper "reached the limit" message. New `tpe_daily_limit_desc` key added and used.

### Changed

- Documentation refresh: **Customize locales** guide rewritten around the new runtime mechanism; **Configuration Files** updated to reflect that `Config.Locale` accepts any language shipped in `locales/`.
- Internal API surface: the 5 balance-changing context actions (`deposit` / `withdraw` / `transfer` / `moveSavings` / `moveToBank`) now return `{ success, error?: string }` instead of a bare boolean. Existing callers that only check `.success` continue to work; new callers can display the specific `error` code via `t(error)`.

---

## [1.0.3] — Quasar Phone Pro & bridge overhaul

### Added

- **Quasar Phone Pro support** — both `nash_banking_phone` and `nash_businessbanking_phone` auto-detect the phone system at startup (`lb-phone` or `qs-smartphone-pro`). LB Phone has priority when both are started. No config changes required.
- **Inventory bridge** — pluggable `Inventory.*` API with auto-detect for `ox_inventory`, `qs-inventory`, `qb-inventory`, and a `custom` mode. See [Custom Inventory](compatibility/custom-inventory.md).
- **`nash_banking:itemRemoved` server event** — normalized hook that fires when a banking item leaves a player's inventory. Auto-wired from `ox_inventory:removedItem`; custom inventories trigger it themselves.
- Documentation: new **Custom Inventory** page under Compatibility.

### Changed

- **Reorganized bridge files** under a top-level `bridge/` folder:
  - `bridge/framework/{framework,client,server}.lua` (was `shared/bridge/framework.lua` + `client/bridge.lua` + `server/bridge.lua`)
  - `bridge/inventory/{config,server}.lua` (new)
  - All 5 bridge files are in `escrow_ignore` — customers can edit them directly to plug in a custom framework or inventory.
- Every server-side `exports.ox_inventory:*` call is now routed through the new `Inventory.*` bridge.
- Documentation refresh: **Compatibility Overview**, **Custom Framework**, **Requirements**, **Inventory Items**, **Phone Apps**.

---

## [1.0.2] — Live balance sync, full currency sweep, interaction fixes

### Fixed

- **Phone balance no longer stays frozen** while the ATM / desktop menu shows the correct value.
  - `nash_banking_phone/client.lua` now sends `{ action = 'balanceUpdate', data = { bank, cash, savings } }` (was `{ type = 'balanceUpdate', bank, cash }`, which the NUI's `useNuiEvent` matcher never accepted).
  - Server now broadcasts `nash_banking:updateBalance` on every money-changing callback: `deposit`, `withdraw`, `transfer` (sender **and** target), `moveSavings`, `moveToBank`, `atmDeposit`, `atmWithdraw`.
- **ATM impossible to use in `ox_lib` and `default` interaction modes** — `GetClosestObjectOfType` was being destructured as two values, but the native returns a single object handle (`0` when nothing is found). Fixed in the 3 call sites (`findNearestATM`, ox_lib branch, default branch) with `local entity = GetClosestObjectOfType(...) ; if entity and entity ~= 0 then …`.
- **`DrawText3D` used before definition** in `atm.lua` (default-mode ATM crashed on approach) — moved the local function declaration above `setupATMInteraction`.
- **Business bank NPC unreachable in `ox_lib` / `default` modes** — added `SetupBusinessOxLib` / `SetupBusinessDefault` + `SetupBusinessInteraction` dispatcher (only `SetupBusinessOxTarget` existed before).
- **`ox_lib` textUI stayed visible under the open bank / business menu** — new global `IsNashMenuOpen()` helper + guard on the interaction loops.
- **ATM `ox_target` `addModel` never removed on resource restart** — added the cleanup in `onResourceStop`.

### Changed

- **Currency is now 100 % dynamic across the desktop UI** — swept every remaining hardcoded `€`: `TransactionItem`, `TransfersPage`, `CryptoPage`, `BusinessPage`, `AdminPage`, `SubscriptionModal`, `CardsModal`, `AnalyticsModal`, `ActionModal`, plus the TPE terminal overlay. A server set to `$` / `USD` now shows the right symbol everywhere.
- The TPE overlay (buyer + seller) receives the current `Config.Currency` through the NUI payload.

---

## [1.0.1] — Tier deposit limits, delivery modes, phone quick actions

### Added

- **`maxDepositPerDay` & `maxDepositAmount`** — per-day deposit limits per subscription tier (Standard / Plus / Premium) in `config.subscriptions.lua`.
- **`Config.Delivery.AllowSameBankDelivery`** — choose between the realistic mode (deliver to a different bank) and the convenient mode (deliver to the same bank the player is standing in).
- **`Config.Admin.DemoMode`** — turn the admin panel into a mock-data view for videos / Tebex screenshots without touching the DB.
- **`Config.PreventIdleCam`** — blocks GTA's native AFK camera while any Nash menu is open.
- **`QuickActions` toggles** on both phone apps — show / hide each button on the home screen (Add money / Move / Transfer / More on personal, Deposit / Withdraw / Buy TPE on business) from `config.lua`.
- **`IconFile` option** on both phone apps — swap the app icon for any `.svg`, `.png`, `.jpg`, `.webp`, or `.gif`.

### Changed

- **Currency became dynamic** on the main UI header, home balance, savings, business, crypto, accounts modal and bank info modal (initial pass — completed in v1.0.2 across all remaining pages and the TPE overlay).
- **Add employee** in business banking accepts a player's server ID directly (resolves to identifier server-side) — the employee's real name shows up instead of `4`.
- **Phone number placeholder + auto-format** aligned with LB Phone style (`555 123 9473`, 3+3+4 grouping).

### Fixed

- **Card pickup** now works on all 3 interaction modes (`ox_target`, `ox_lib`, `default`) — press G at the bank where you ordered a card.
- **Card creation failures** now show an explicit toast (e.g. *"You have reached the maximum number of cards for your subscription"*) instead of silently doing nothing.

---

## [1.0.0] — Initial release

The first public release of Nash Banking.

### Banking UI

- Personal banking dashboard with 5 navigable pages: **Home**, **Savings**, **Transfers**, **Crypto**, **Invest**
- **Profile page** with personal info, custom avatar (URL) and subscription management
- **Accounts modal** — consolidated view of every balance (main bank, cash, savings, per-card subtotals)
- **Analytics modal** — deposits / withdrawals stats with charts
- **Bank Info modal** — banking information for the player
- **ATM map** — interactive Leaflet map showing every bank branch and ATM, with searchable sidebar and GPS-waypoint integration

### Card system

- Three card tiers — **Standard**, **Plus**, **Premium** — each with its own gradient and limits
- **Virtual** and **physical** card formats, with optional delivery point pickup
- Per-card freeze / unfreeze
- PIN management (with 3-strikes auto-freeze on the ATM)
- Player-defined monthly spending limit per card (toggle in config: `BySubscription = false`)
- **Per-card transaction history / metadata** — each card tracks its own activity
- Physical card item (`nash_card_physical`) with inspect overlay when used from inventory
- Automatic freeze on item loss (configurable)

### Realistic ATM

- Detects vanilla GTA Fleeca ATMs + supports custom standalone ATM locations
- Card insertion via wallet UI (pick which card to use)
- **3D DUI screen** rendered on the ATM prop with a working PIN keypad
- Withdraw / Deposit flow with per-card daily limits
- Per-month free withdrawal count from the subscription tier

### Player-to-Player TPE

- TPE item (`nash_tpe`) — seller takes the terminal, enters the amount, hands it to the buyer
- Buyer picks a card in their wallet, drag & drop on the TPE to confirm
- Notifications + animations on both sides
- Configurable 2% transaction fee for the seller
- Business-linked TPEs route income to the business balance (with optional society sync for job-linked businesses)

### Pluggable shop integration

- `exports.nash_banking:CardPayment(source, amount, description)` — any shop / job / payment script on the server can route card payments through Nash Banking
- Daily and single-transaction spending limits enforced server-side
- Transactions appear in the player's banking history with the calling script's description
- Includes a working integration with `nash_shops` as a reference

### Business banking

- Per-business balance and transaction ledger
- Three roles: Boss, Manager, Employee (configurable `AccessLevel`)
- Personal businesses (player-created) and job-linked businesses (admin-created, tied to a framework job)
- Boss can purchase a business-routed TPE
- Add / remove employees by player server ID (auto-resolves to identifier)

### Subscriptions

- Three tiers — Standard (free), Plus (500€ / 30 days), Premium (2000€ / 30 days)
- Each tier unlocks: daily transfer / withdraw limits, max amounts, max number of cards, NashPoints multiplier, savings rate, Invest access, Crypto access, priority support
- Auto-renewal with configurable `ChargeInterval` and downgrade-on-fail behavior
- Live subscription comparison modal in the Profile page

### Savings

- Configurable interest rate **per subscription tier**
- Periodic interest (default: hourly cycle)
- Offline catch-up on reconnection (cap configurable, default 7 days)
- Dedicated Savings page in the UI

### Invest (stocks / ETFs)

- 12 default tickers (NVDA, AAPL, TSLA, NFLX, AMZN, MSFT, SPY, QQQ, VTI, ARKK, IVV, VOO)
- Mean-reversion price engine with configurable volatility
- Live charts per asset
- Buy / sell / portfolio
- Subscription-gated (Plus+ by default)

### Crypto

- 8 default pairs (BTC, ETH, SOL, XRP, DOGE, ADA, DOT, AVAX)
- Faster update cadence than stocks
- Same buy / sell / portfolio flow
- Subscription-gated (Premium-only by default)

### Messaging

- In-bank chat between players (text + GIF)
- Giphy integration with searchable picker
- Per-conversation history with swipeable actions
- Server-side rate-limited

### Admin panel

- Command: `/bankadmin` (configurable)
- Global stats — money in circulation, total businesses, registered players
- Top players and top businesses leaderboards
- Player search (by server ID, identifier or name)
- Add / Remove money for any player or business
- Create and delete businesses
- **Demo Mode** flag for promotional video / screenshots (returns mock data without touching the DB)

### LB Phone companion apps

- **Personal app** — home, cards, transfers (with messaging) in a phone-optimized UI
- **Business app** — balance, employees, transactions
- Both apps fully localized (FR / EN)
- Configurable app icon, name, and default-installed flag

### Multi-framework support

- Built-in bridge — auto-detects ESX, QBCore, or QBOX
- Single API used internally by every server / client script (`Bridge.*`)
- Documentation includes a guide for wiring a custom framework

### Localization

- French and English shipped by default
- Two parallel locale systems (Lua + React) for full UI coverage
- All notifications, transaction descriptions, target labels, and UI strings are translatable

### Discord webhook logging

- 8 separate categories (transactions, ATM, cards, subscriptions, business, admin, TPE, invest)
- One webhook per category — split the load across multiple channels
- Per-event-type embed colors

### Technical

- Performance indexes on every hot table (`nash_transactions.idx_daily_spend`, `nash_cards.idx_card_active`, etc.)
- Configurable transaction auto-purge (default: 90 days)
- Streamed bank card and TPE props (no extra resource required)
- DUI-based 3D ATM screen
- Native idle-camera blocker while menus are open (configurable)
- Rate-limited server events on every sensitive endpoint (2 second cooldown per player)
