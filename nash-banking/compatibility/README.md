# Compatibility Overview

Nash Banking uses two independent bridges — one for the framework, one for the inventory — with auto-detect at startup and a `'custom'` escape hatch for both. A third bridge handles phone-app detection.

## Supported out of the box

| Bridge | Auto-detected |
|---|---|
| **Framework** | `es_extended` (ESX) · `qbx_core` (QBOX) · `qb-core` (QBCore) |
| **Inventory** | `ox_inventory` · `qs-inventory` · `qb-inventory` |
| **Phone** (optional) | `nash_phone` · `lb-phone` · `qs-smartphone` · `qs-smartphone-pro` |

## Bridge architecture

```
bridge/
  framework/
    framework.lua   -- detect + FrameworkObj wiring
    client.lua      -- Bridge.* wrappers (client)
    server.lua      -- Bridge.* wrappers (server)
  inventory/
    config.lua      -- Config.Inventory + Config.CustomInventory stub
    server.lua      -- Inventory.* wrappers
```

Every server / client script uses `Bridge.*` and `Inventory.*` instead of calling ESX / QBCore / ox_inventory directly — the bridges dispatch to the right implementation at runtime.

All five files above are in `escrow_ignore` — you can edit them on your server for any custom setup.

## Framework detection

`bridge/framework/framework.lua` runs on both client and server at resource start:

1. If `Config.Framework` is `'esx'`, `'qbcore'` or `'qbox'`, use it directly (skip detection).
2. Otherwise (`'auto'`, default):
   1. `es_extended` started → `esx`
   2. `qbx_core` started → `qbox`
   3. `qb-core` started → `qbcore`
3. If none is started yet, the same checks are run against `GetResourceState() ~= 'missing'` as a fallback.
4. If no framework is found → prints `[Nash Banking] No supported framework detected`.

```lua
-- shared/config.lua
Config.Framework = 'auto' -- 'auto' | 'esx' | 'qbcore' | 'qbox'
```

Supported out of the box: [ESX](esx.md) · [QBCore](qbcore.md) · [QBOX](qbox.md).

## Inventory detection

`bridge/inventory/server.lua` runs on the server at resource start:

1. If `Config.Inventory` is set to a specific inventory, use it directly.
2. Otherwise (`'auto'`, default):
   1. `ox_inventory` started → `ox_inventory`
   2. `qs-inventory` started → `qs-inventory`
   3. `qb-inventory` started → `qb-inventory`
3. If none are started → falls back to `'custom'` (dispatches to `Config.CustomInventory`).

```lua
-- bridge/inventory/config.lua
Config.Inventory = 'auto' -- 'auto' | 'ox_inventory' | 'qs-inventory' | 'qb-inventory' | 'custom'
```

## Phone detection

The two optional phone apps (`nash_banking_phone`, `nash_businessbanking_phone`) auto-detect `nash_phone` first, then `lb-phone`, `qs-smartphone` (Quasar V3) and `qs-smartphone-pro`. See [Phone Apps](../installation/lb-phone.md).

## Extending the bridges

If you use a framework or inventory the auto-detect doesn't cover:

- [Custom Framework](custom-framework.md) — wire your own framework into `bridge/framework/*`.
- [Custom Inventory](custom-inventory.md) — fill in `Config.CustomInventory` with your inventory's API.
