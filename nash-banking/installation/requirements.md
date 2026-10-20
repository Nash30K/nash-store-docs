# Requirements

## Server software

- **FiveM artifacts** — latest recommended build.
- **MySQL** — 5.7+ / MariaDB 10.4+.

## Required resources

| Resource | Purpose |
| -------- | ------- |
| `ox_lib` | UI, utilities, locales |
| `oxmysql` | Database access |
| An **inventory** | Physical card & TPE items — `ox_inventory`, `qs-inventory`, `qb-inventory`, or any custom inventory via the [inventory bridge](../compatibility/custom-inventory.md) |
| A **framework** | `es_extended` (ESX), `qb-core` (QBCore), `qbx_core` (QBOX), or any custom framework via the [framework bridge](../compatibility/custom-framework.md) |

## Optional resources

| Resource | Purpose |
| -------- | ------- |
| `ox_target` | Cleanest interaction mode for bank / ATM / business NPCs. `ox_lib` textUI and default 3D text are also supported — configurable in `shared/config.lua` |
| A **phone** | Enables the two banking phone apps — `nash_phone`, `lb-phone`, `qs-smartphone` or `qs-smartphone-pro` (auto-detected) |

## Compatibility matrix

See the [Compatibility](../compatibility/README.md) section for the framework / inventory / phone bridge overview and custom-mode guides.
