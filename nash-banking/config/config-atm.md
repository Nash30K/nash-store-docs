# config.atm.lua

ATM detection, PIN security and the ATM map UI (styling of the "Find ATMs" view in the banking NUI).

File: `shared/config.atm.lua`

## Core ATM behavior

```lua
Config.ATM = {
    Enabled = true,
    Models = {
        'prop_atm_01',
        'prop_atm_02',
        'prop_atm_03',
        'prop_fleeca_atm',
    },
    MaxPinAttempts = 3,           -- PIN tries before the card freezes
    FreezeOnMaxAttempts = true,
    ScanRadius = 50.0,            -- ATM prop scan radius (meters)
    InteractionDistance = 1.5,

    -- Toggles for individual ATM actions.
    -- false = hide the button from the ATM menu and refuse the callback server-side.
    AllowDeposit = true,
    AllowWithdraw = true,
}
```

## Disable specific ATM actions

Set `AllowDeposit` or `AllowWithdraw` to `false` to remove the corresponding button from the ATM menu. Remaining buttons shift up in place, so a menu with `AllowDeposit = false` renders as **Withdraw / Balance / Exit**.

Both options are enforced on the server as well, so a tampered client cannot bypass the restriction. Useful setups:

- **Deposits at the bank only** — `AllowDeposit = false` forces players to visit an agency to add cash to their account. ATMs remain usable for withdrawals and balance checks.
- **Bank-agency roleplay** — set both to `false` to turn ATMs into balance-only kiosks (Balance / Exit).

## Daily limits per subscription

Daily ATM limits are defined per tier in [`config.subscriptions.lua`](config-subscriptions.md) under `features.maxWithdrawPerDay` and `features.maxWithdrawAmount`. `-1` means unlimited.

Free monthly withdrawals come from `features.freeAtmWithdrawals`.

## ATM map UI

The banking NUI ships with a "Find ATMs" tab rendering Banks + ATMs on an interactive map. All styling is configurable.

```lua
Config.ATMMap = {
    Enabled = true,

    Title = 'ATMs & Banks',
    SearchPlaceholder = 'Search...',
    LoadingText = 'Loading map...',

    BankPopupLabel = 'Bank branch',
    ATMPopupLabel = 'ATM',
    GPSButtonLabel = 'Set waypoint',

    -- Marker colors (CSS gradients)
    BankMarkerColor = 'linear-gradient(135deg, #7c3aed, #a855f7)',
    BankMarkerShadow = 'rgba(124, 58, 237, 0.5)',
    ATMMarkerColor = 'linear-gradient(135deg, #2563eb, #3b82f6)',
    ATMMarkerShadow = 'rgba(37, 99, 235, 0.5)',
    PlayerMarkerColor = '#22c55e',

    -- Legend
    BankLegendColor = '#a855f7',
    ATMLegendColor = '#3b82f6',
    PlayerLegendColor = '#22c55e',
    BankLegendLabel = 'Bank',
    ATMLegendLabel = 'ATM',
    PlayerLegendLabel = 'You',

    GPSButtonGradient = 'linear-gradient(135deg, #7c3aed, #a855f7)',

    BankMarkerSize = 32,
    ATMMarkerSize = 28,

    DefaultZoom = 1,
    MinZoom = -2,
    MaxZoom = 5,
    FlyToZoom = 3,
}
```

See [Guides › Customize the ATM map](../guides/customize-atm.md) for concrete theming examples.
