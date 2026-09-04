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
| **Inventory** | `ox_inventory` · `qb-inventory` / `lj-inventory` · the framework's own | `Config.Inventory` | `'auto'` |

Both keys live in `config/bridge.lua`. In `'auto'` mode you normally have nothing to do: the
console announces what was picked at startup.

## Server-side only

There is no framework code in `client/`. `client/bridge/nui.lua` is the NUI router, not a
framework bridge: it forwards NUI actions to server callbacks and has nothing to do with
ESX, QBCore or QBOX. Everything framework-related is in `server/bridge/`, which means a
custom framework never requires a client-side change.

## Files

| File | Role |
| ---- | ---- |
| `config/bridge.lua` | The two switches, `Config.Framework` and `Config.Inventory` |
| `server/bridge/framework.lua` | The `Framework.*` interface, the detection order, the fallback contract |
| `server/bridge/esx.lua` | ESX provider |
| `server/bridge/qbcore.lua` | QBCore provider |
| `server/bridge/qbox.lua` | QBOX provider |
| `server/bridge/inventory.lua` | The `Inventory.*` interface and its three built-in providers |
| `server/bridge/custom.lua` | The only file meant to be edited: custom framework **and** custom inventory |

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
[nash_phone] AUCUN framework détecté. Le téléphone ne pourra identifier personne.
[nash_phone] Si vous utilisez un framework maison, renseignez Config.Framework et remplissez server/bridge/custom.lua.
```

{% hint style="warning" %}
The detection order is a hard-coded list of three names. Registering your own
`NashFw.myframework` with a working `detect()` is **not** enough: `'auto'` will never try
it. Name it in `Config.Framework`. See [Custom Framework](custom-framework.md).
{% endhint %}

## Inventory detection

With `Config.Inventory = 'auto'`, providers are tried in this order:

1. `ox`: `ox_inventory` started
2. `qb`: `qb-inventory` **or** `lj-inventory` started
3. `framework`: always accepts; reads the framework's native inventory

```
[nash_phone] inventaire détecté : ox.
```

`ox_inventory` comes first because when it runs, it holds the items regardless of the
framework underneath. The `framework` provider is the last resort and always accepts, so
`'auto'` never ends with "no inventory".

Accepted values: `'auto'`, `'ox'`, `'qb'`, `'framework'`, `'custom'`. Anything else:

```
[nash_phone] Config.Inventory = "qs" : inconnu. Attendu : auto, ox, qb, framework, custom.
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
| `addBank`, `removeBank` | `false` |
| `players` | `{}` |

This is what lets a half-wired custom bridge still boot instead of throwing a Lua error on
every callback. The one function with no survivable neutral value is `identifier`: the
bootstrap callback returns `false` without it, so the phone never opens.

## What the code guarantees, and what it does not

The three bridges implement the same interface and every app goes through it: there is not a
single ESX, QBCore or QBOX call outside `server/bridge/`. That is a structural guarantee, not
a testing claim.

{% hint style="warning" %}
ESX is the reference platform: it is the framework named in the resource's own required-
dependency list, and the two SQL helpers shipped in `sql/` (`item_phone.sql`,
`item_earbuds.sql`) target the default ESX `items` table. The QBCore and QBOX bridges are
complete and use only documented core APIs, but we do not claim production hours on them.
On a QBCore or QBOX server, run the checklist at the bottom of
[QBCore](qbcore.md) / [QBOX](qbox.md) before opening.
{% endhint %}

`qs-inventory` is **not** referenced anywhere in the code and is not auto-detected. It has to
go through [Custom Inventory](custom-inventory.md).

## Changing framework after launch

{% hint style="danger" %}
Every phone table is keyed on the identifier the framework hands out, in a `VARCHAR(64)`
`owner` column: an ESX license, a QB `citizenid`. Those do not have the same shape. Switching
framework on a live server leaves every player with an empty phone: numbers, contacts,
messages, photos and social accounts all become orphaned rows. Decide before opening, or write
a migration for it.
{% endhint %}

## Diagnostics

Two dev commands report what the bridges resolved to. They require `Config.Debug = true` in
`config/main.lua`, plus a framework group listed in `Config.DebugGroups` (default
`{ 'admin', 'superadmin' }`) or the `nash_phone.debug` ACE. The server console always passes.

| Command | What it shows |
| ------- | ------------- |
| `/phonedeps` | The framework and inventory actually in use, and what stops working without each dependency |
| `/phonecheck` | Short verdict: including "the phone item does not exist, nobody can open the phone" |

## Going further

- [ESX](esx.md)
- [QBCore](qbcore.md)
- [QBOX](qbox.md)
- [Custom Framework](custom-framework.md)
- [Custom Inventory](custom-inventory.md)
