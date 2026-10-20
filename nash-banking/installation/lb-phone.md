# Phone Apps (NASH Phone, LB Phone, Quasar Phone) - optional

Nash Banking ships two phone apps:

- **`nash_banking_phone`** — personal banking app
- **`nash_businessbanking_phone`** — business banking app

Both are independent resources that depend on `nash_banking` being started first (they import `@nash_banking/bridge/framework/framework.lua` and `@nash_banking/bridge/framework/client.lua`).

## Requirements

- One phone resource started: **`nash_phone`**, **`lb-phone`**, **`qs-smartphone`** (Quasar V3) or **`qs-smartphone-pro`**
- `nash_banking` started before the phone apps

The phone type is **auto-detected at startup**, no config needed. Detection order: `nash_phone` → `lb-phone` → `qs-smartphone` → `qs-smartphone-pro`.

## Installing an app

1. Drop `nash_banking_phone/` (and/or `nash_businessbanking_phone/`) inside your `resources/` folder.
2. Add to your `server.cfg` **after** `nash_banking`:

    ```cfg
    ensure nash_phone                   # or lb-phone, qs-smartphone, qs-smartphone-pro
    ensure nash_banking
    ensure nash_banking_phone
    ensure nash_businessbanking_phone
    ```

3. Restart the server. In-game, open the phone — the app should be visible.

## How the apps register

Each app auto-detects which phone system is running and uses the appropriate API. Console print at startup:

```
[Nash Banking Phone] Phone system: lb
```
…or:
```
[Nash Banking Phone] Phone system: quasar
```
…or:
```
[Nash Banking Phone] Phone system: nash
```

- **NASH Phone** - declared via `exports['nash_phone']:AddCustomApp({...})`. With `DefaultApp = true` the app is installed and placed at the end of the last home page, once per character. It is also listed in the App Store under "Server Apps". If a player removed it from the home screen, it stays in the App Library.
- **LB Phone** - pre-installed by default via `exports['lb-phone']:AddCustomApp({...})`.
- **Quasar Phone V3** - registered via `exports['qs-smartphone']:addCustomApp({...})`.
- **Quasar Phone Pro** - registered in the phone's app store via `exports['qs-smartphone-pro']:addCustomApp({...})`.

The app re-registers automatically if the phone resource restarts (listener on `onResourceStart`).

## Configuration

Each phone app has its own `config.lua`. **The two apps use slightly different shapes** — the personal app wraps everything in a `Config.Phone` table, the business app uses flat `Config.*` keys. The fields themselves are equivalent.

### Personal — `nash_banking_phone/config.lua`

```lua
Config.Phone = {
    Locale       = 'en',          -- 'fr' or 'en'
    AppName      = 'Nash Banking',
    DefaultApp   = true,          -- NASH Phone / LB Phone: pre-installed (false = downloadable from the app store)
    Currency     = '€',
    CurrencyCode = 'EUR',
    BankName     = 'Nash Banking',
    IconFile     = 'icon.svg',    -- file in ui/assets/ (see "App icon" below)
}
```

### Business — `nash_businessbanking_phone/config.lua`

```lua
Config.Locale       = 'en'
Config.AppName      = 'Nash Business'
Config.DefaultApp   = true
Config.Currency     = '€'
Config.CurrencyCode = 'EUR'
Config.BankName     = 'Nash Business Banking'
Config.IconFile     = 'icon.svg'   -- file in ui/assets/ (see "App icon" below)
```

## App icon

The icon shown on the phone home screen is loaded from `ui/assets/<IconFile>` inside each phone resource. To change it, drop a new file (`.svg`, `.png`, `.jpg`, `.webp`, or `.gif`) in `ui/assets/` and point `IconFile` at it:

```lua
-- Personal phone
Config.Phone.IconFile = 'my_logo.png'

-- Business phone
Config.IconFile = 'my_logo.png'
```

Then restart the phone resource. Recommended size: 120×120 px or larger, square, transparent background if possible — the phone scales the icon down on the home screen.

## How the phone talks to the server

Both phones use standard FiveM NUI primitives (`RegisterNUICallback` on the server, `fetch()` from the iframe), so the UI code is identical across NASH Phone, LB Phone and Quasar Phone. On NASH Phone the app does not need the `nash-bridge.js` script: it calls its own NUI callbacks directly and listens to the raw `{ action, data }` messages the phone relays. The React app reuses the exact same server callbacks as the desktop UI via `Bridge.TriggerCallback`:

```lua
-- Example: home screen data
Bridge.TriggerCallback('nash_banking:getTransactions', function(tx, registered, info, savings, rate)
    ...
end)
```

Any feature added to `server/main.lua` is immediately usable in the desktop NUI **and** both phone systems.

## Disabling the apps

- Don't `ensure` the app resource in `server.cfg`, **or**
- Set `Config.LBPhone.Enabled = false` in `nash_banking/shared/config.lua` to hide the "open in phone" prompts in the desktop UI.

## Disabling phone notifications

In-bank events (transfers received, TPE paid, requests) can mirror to phone notifications, on NASH Phone (`SendNotification`) and LB Phone. Toggle with `Config.PhoneNotifications = false` in [`shared/config.lua`](../config/config-main.md).
