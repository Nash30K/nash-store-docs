# Compatibility Overview

NASH Phone reaches your server through **two independent bridges**: one for the framework,
one for the inventory. Both are resolved at runtime, both auto-detect, and both have a
`'custom'` escape hatch.

They are chosen **separately**, and that is the point. An ESX server running `ox_inventory`
is the most common setup there is: asking ESX whether a player carries the phone item would
give the wrong answer, because ESX no longer holds the items. `Config.Framework` and
`Config.Inventory` are two different switches.

## Supported out of the box

| Bridge | Auto-detected | Config key | Default |
| ------ | ------------- | ---------- | ------- |
| **Framework** | `es_extended` (ESX) · `qb-core` (QBCore) · `qbx_core` (QBOX) | `Config.Framework` | `'auto'` |
| **Inventory** | `ox_inventory` · `qs-inventory` · `qb-inventory` / `ps-inventory` / `lj-inventory` · the framework's own | `Config.Inventory` | `'auto'` |

Both keys live in `config/bridge.lua`. In `'auto'` mode you normally have nothing to do: the
console announces what was picked at startup.

## Mostly server-side

Almost everything framework-related is in `server/bridge/`. The client knows the framework in
two places only:

| What | Events | Why |
| ---- | ------ | --- |
| Death state | `QBCore:Player:SetPlayerData`, `qbx_core:client:onSetMetaData`, `QBCore:Client:OnPlayerLoaded` / `OnPlayerUnload`, plus one read of the player data from `qbx_core` or `qb-core` when the player loads | The `isdead` and `inlaststand` metadata of QBCore and QBOX, see [Death](#death-closes-the-phone) |
| Character switch | `esx:onPlayerLogout`, `QBCore:Client:OnPlayerUnload`, `qbx_core:client:playerLoggedOut` | Closes the phone when a player goes back to character selection. The phone's own server also sends `nash_phone:playerLogout` from the bridge's unload event, so a custom framework is covered too |

A custom framework never requires a client-side change: on a framework where those events do not
exist, they simply never fire.

## Files

| File | Role |
| ---- | ---- |
| `config/bridge.lua` | The two switches, `Config.Framework` and `Config.Inventory` |
| `server/bridge/framework.lua` | The `Framework.*` interface, the detection order, the fallback contract |
| `server/bridge/esx.lua` | ESX provider |
| `server/bridge/qbcore.lua` | QBCore provider |
| `server/bridge/qbox.lua` | QBOX provider |
| `server/bridge/inventory.lua` | The `Inventory.*` interface and its four built-in providers |
| `server/bridge/custom.lua` | The only bridge file meant to be edited: custom framework **and** custom inventory |

The resource is protected by Asset Escrow: `server/bridge/custom.lua` is the only bridge file
that stays readable and editable (along with `config/*.lua`). The other `server/bridge/` files
of the table are encrypted, and listed here only so the console messages make sense.

`fxmanifest.lua` loads `server/bridge/**.lua` **before** `server/main.lua`, because
`server/main.lua` subscribes to player loading as soon as it is loaded.

## How a provider registers

Each provider file drops a table into the global `NashFw` and calls nothing at load time:

```lua
NashFw = NashFw or {}
NashFw.myframework = {
    playerLoadedEvent = 'myframework:playerLoaded',
    detect = function() return GetResourceState('my_core') == 'started' end,
    build  = function() return { identifier = ..., fullName = ... } end,
}
```

`build()` is called at the **first actual use**, not at resource start. That is why
`server/bridge/esx.lua` can ship on a QBCore server without breaking anything: nothing calls
`getSharedObject()` until an ESX server actually needs it. File load order inside
`server/bridge/**.lua` is irrelevant for the same reason.

## Framework detection

With `Config.Framework = 'auto'`, providers are tried in this fixed order:

1. `qbox`: `GetResourceState('qbx_core') == 'started'`
2. `qbcore`: `GetResourceState('qb-core') == 'started'`
3. `esx`: `GetResourceState('es_extended') == 'started'`

{% hint style="info" %}
QBOX is tested **before** QBCore on purpose. Many QBOX servers run a `qb-core` compatibility
resource alongside `qbx_core`; checking QBCore first would file them under the wrong bridge
and make them miss their own exports.
{% endhint %}

On success the console prints, in French:

```
[nash_phone] framework détecté : qbox.
```

Forcing it skips detection entirely:

```lua
-- config/bridge.lua
Config.Framework = 'esx'  -- 'auto' | 'esx' | 'qbcore' | 'qbox' | 'custom'
```

```
[nash_phone] framework : esx (imposé par la configuration).
```

A name that matches no registered provider prints the list of the ones that exist:

```
[nash_phone] Config.Framework = "qbcore2" : aucun pont de ce nom. Ponts disponibles : custom, esx, qbcore, qbox
```

And when nothing at all is found:

```
[nash_phone] AUCUN framework détecté pour l'instant. Le téléphone ne pourra identifier personne.
[nash_phone] Démarrez le core AVANT nash_phone. Framework maison : renseignez Config.Framework et remplissez server/bridge/custom.lua.
```

A failed detection is **not** remembered. If `nash_phone` starts before the core in your
`server.cfg`, the phone asks again on every call and works as soon as the core is started,
without a restart. The usable items are only registered once a framework is found (it waits up
to 60 seconds for it at startup), so start the core first all the same.

{% hint style="warning" %}
The detection order is a hard-coded list of three names. Registering your own
`NashFw.myframework` with a working `detect()` is **not** enough: `'auto'` will never try
it. Name it in `Config.Framework`. See [Custom Framework](custom-framework.md).
{% endhint %}

## Inventory detection

With `Config.Inventory = 'auto'`, providers are tried in this order:

1. `ox`: `ox_inventory` started
2. `qs`: `qs-inventory` started
3. `qb`: `qb-inventory`, `ps-inventory` **or** `lj-inventory` started
4. `framework`: always accepts; reads the framework's native inventory

```
[nash_phone] inventaire détecté : ox.
```

`ox_inventory` comes first because when it runs, it holds the items regardless of the
framework underneath. `qs-inventory` comes before `qb-inventory` because on QBCore it replaces
it. The `framework` provider is the last resort and always accepts, so `'auto'` never ends
with "no inventory".

Accepted values: `'auto'`, `'ox'`, `'qs'`, `'qb'`, `'framework'`, `'custom'`. Anything else:

```
[nash_phone] Config.Inventory = "codem" : inconnu. Attendu : auto, ox, qs, qb, framework, custom.
```

The phone asks an inventory two questions and nothing more:

| Question | Interface | Used for |
| -------- | --------- | -------- |
| Does this player carry this item? | `Inventory.hasItem(src, item)` | Opening the phone, pairing the earbuds |
| Is this item **declared** on the server? | `Inventory.itemExists(name)` | Startup warning only |

`itemExists` is deliberately **three-valued**: `true`, `false`, or `nil` when the inventory in
place cannot answer (not loaded yet, export missing). On `nil` the phone stays quiet rather
than sending an owner chasing a problem that may not exist.

## The degradation contract

Every `Framework.*` call goes through a wrapper. A provider that does not implement a function
returns a neutral value and prints **one** warning, once:

```
[nash_phone] le pont « custom » n'implémente pas getBank() : valeur neutre rendue.
```

| Interface | Neutral value when unimplemented |
| --------- | -------------------------------- |
| `getPlayer`, `identifier`, `group`, `fullName`, `job`, `usableItem` | `nil` |
| `getBank`, `getCash` | `0` |
| `addBank`, `removeBank`, `setPhoneNumber` | `false` |
| `players` | `{}` |

This is what lets a half-wired custom bridge still boot instead of throwing a Lua error on
every callback. The one function with no survivable neutral value is `identifier`: the
bootstrap callback returns `false` without it, so the phone never opens.

## What the code guarantees, and what it does not

The three bridges implement the same interface and every app goes through it: there is not a
single ESX, QBCore or QBOX call outside `server/bridge/`, apart from the client events listed in
[Mostly server-side](#mostly-server-side). That is a structural guarantee, not a testing claim.

## Character switch

On a multi-character server, a player can go back to character selection without
disconnecting. The phone treats it like a departure: the call in progress ends, an InstaPic
live stream stops, the player leaves duty in Services, the music stops, the number stops
ringing this player, and a byCloud session is closed so the next character never reads the
previous one's mailbox. The signal is `esx:playerLogout` on ESX and
`QBCore:Server:OnPlayerUnload` on QBCore and QBOX, or `playerUnloadedEvent` on a
[custom framework](custom-framework.md#playerunloadedevent).

On the client, the phone closes and locks, and **saves nothing for any character** until the
next one is loaded: home layout, installed apps, settings, wallpaper, profile, Notes, Calendar,
Reminders, contact photos, screen time, the byCloud keychain backup and the first-run setup.
A change still waiting to be sent at that moment is dropped rather than written into the next
character, and a server reply that arrives after the switch is ignored. The next character's
load reloads their own layout, apps, settings, app data and screen time.

## Death closes the phone

The rule is the same on every framework: a dead player cannot open the phone, an open phone
closes, and an incoming call does not open it. ESX keeps a dead player dead, which the game
sees. QBCore and QBOX revive the character and play a death animation instead, so the phone
also reads their `isdead` and `inlaststand` metadata: a player on the ground, even in last
stand, cannot use the phone.

{% hint style="warning" %}
ESX is the reference platform: the two SQL helpers shipped in `sql/` (`item_phone.sql`,
`item_earbuds.sql`) target the default ESX `items` table. No framework is listed in the
resource's hard dependencies (only OneSync, `ox_lib` and `oxmysql` are). The QBCore and QBOX bridges are
complete and use only documented core APIs, but we do not claim production hours on them.
On a QBCore or QBOX server, run the checklist at the bottom of
[QBCore](qbcore.md) / [QBOX](qbox.md) before opening.
{% endhint %}

`qs-inventory` has its own provider, auto-detected, built on the exports documented by Quasar.
Other inventories go through the framework's native inventory when they replace it, or through
[Custom Inventory](custom-inventory.md).

## Changing framework after launch

{% hint style="danger" %}
Every phone table is keyed on the identifier the framework hands out, in a `VARCHAR(64)`
`owner` column: an ESX license, a QB `citizenid`. Those do not have the same shape. Switching
framework on a live server leaves every player with an empty phone: numbers, contacts,
messages, photos and social accounts all become orphaned rows. Decide before opening, or write
a migration for it.
{% endhint %}

## Diagnostics

The server console tells you what the bridges resolved to. Read it after a restart:

| What to look for | Meaning |
| ---------------- | ------- |
| `framework détecté : ...` or `framework : ... (imposé par la configuration).` | The framework bridge in use |
| `inventaire détecté : ...`, `inventaire : ... (imposé par la configuration).` or `inventaire : pont personnalisé.` | The inventory bridge in use |
| A red block starting with `Config.UseItem = true but the item "phone" DOES NOT EXIST` | The inventory in place says `Config.ItemName` is not declared: nobody can open the phone |

The red block names your own `Config.ItemName` (`phone` by default). It is printed about 15
seconds after the framework is found, and only with `Config.UseItem = true`. When the inventory
cannot answer (`itemExists` returns `nil`), nothing is printed.

## Going further

- [ESX](esx.md)
- [QBCore](qbcore.md)
- [QBOX](qbox.md)
- [Custom Framework](custom-framework.md)
- [Custom Inventory](custom-inventory.md)
