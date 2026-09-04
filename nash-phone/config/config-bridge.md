# config/bridge.lua

Framework and inventory selection.

File: `config/bridge.lua`, loaded as a shared script right after `config/main.lua`.

The phone talks to your server through two bridges, and they are chosen **separately**. An ESX
server running `ox_inventory` is the most common setup there is, so a single combined setting
would be wrong for most servers.

In `'auto'` mode there is usually nothing to do. The console prints one line per bridge at
startup:

```
[nash_phone] framework détecté : qbox.
[nash_phone] inventaire détecté : ox.
```

If those two lines say what you expect, this file does not concern you.

## Config.Framework

```lua
Config.Framework = 'auto'
```

| Value | Meaning |
| ----- | ------- |
| `'auto'` | Automatic detection (default, recommended) |
| `'esx'` | ESX Legacy |
| `'qbcore'` | QBCore |
| `'qbox'` | Qbox (`qbx_core`) |
| `'custom'` | Your own, written in `server/bridge/custom.lua` |

Automatic detection tries **qbox, then qbcore, then esx**. Qbox is tested before QBCore on
purpose: many Qbox servers run a compatibility `qb-core` resource alongside, and testing
QBCore first would file them under the wrong bridge and make the phone miss Qbox's own
exports.

Forcing a value prints a different line, so you can tell detection from a manual choice:

```
[nash_phone] framework : esx (imposé par la configuration).
```

A name that matches no bridge is refused, and the console lists the bridges that do exist:

```
[nash_phone] Config.Framework = "qb" : aucun pont de ce nom. Ponts disponibles : custom, esx, qbcore, qbox
```

The phone then starts with no framework, and identifies nobody.

{% hint style="danger" %}
**Changing framework on a live server orphans the data.** Every phone table is indexed on the
identifier the framework hands out, and that identifier does not have the same shape from one
framework to the next (an ESX license, a QB citizenid). Players would find an empty phone, and
their numbers, contacts, messages and photos would become unreachable. Decide this before the
server opens, or write a migration for it.
{% endhint %}

## Config.Inventory

```lua
Config.Inventory = 'auto'
```

| Value | Meaning |
| ----- | ------- |
| `'auto'` | Automatic detection (default, recommended) |
| `'ox'` | `ox_inventory` |
| `'qb'` | `qb-inventory` and its derivatives |
| `'framework'` | The framework's own inventory (ESX, QBCore) |
| `'custom'` | Your own, written in `server/bridge/custom.lua` |

Automatic detection tries **ox, then qb, then framework**. `ox_inventory` comes first because
when it is running it is the resource that actually holds the items, whatever framework sits
underneath.

The phone asks an inventory only two questions: does this player own the phone item, and is
that item declared on the server. Nothing else. Both are used by `Config.UseItem` /
`Config.ItemName` in [config/main.lua](config-main.md).

## Custom bridge

Set `Config.Framework = 'custom'` and/or `Config.Inventory = 'custom'`, then fill
`server/bridge/custom.lua` and restart the resource. That file is the only one in the phone
meant to be edited: everything else can be updated without touching it.

`server/bridge/custom.lua` never self-detects. It is used only when you name it here.

### Framework functions (`NashFw.custom`)

Only `identifier` is mandatory. Any function you leave empty returns a neutral value and
writes one console warning, so the phone still boots and you can wire things up in stages.

| Function | Required | What you lose if left empty |
| -------- | -------- | --------------------------- |
| `identifier(src)` | **Yes** | Nothing works. This is the key of every phone table |
| `fullName(src)` | No | The RP name shows as "Unknown" (translated) everywhere |
| `getPlayer(src)` | No | Only passed back to other functions of the same bridge |
| `group(src)` | No | No admin command is ever granted |
| `job(src)` | No | The Services app recognises no employee |
| `getBank(src)` / `getCash(src)` | No | The Wallet app shows 0 |
| `addBank(src, amount)` / `removeBank(src, amount)` | No | Transfers fail |
| `players()` | No | Returns an empty list |
| `usableItem(name, cb)` | No | The phone no longer opens from the item, only from the key |
| `invHasItem(src, item)` | No | Only used when `Config.Inventory = 'framework'` |
| `invItems()` | No | The item-declaration check cannot run |

`job(src)` must return exactly this shape, or `nil`:

```lua
{ name = 'police', grade = 3, gradeName = 'sergeant', label = 'Police', onDuty = true }
```

`onDuty` is optional. It is only read when you set `Config.Services.ownDuty = false` to drive
duty from your own system (see [config/services.lua](config-services.md)).

`NashFw.custom.playerLoadedEvent` names the event your framework fires when a character is
loaded. Leave it `nil` if you have none: the phone catches up on the first NUI start instead,
a little later.

{% hint style="warning" %}
`identifier` must be **stable over time** for a given character. It is what ties a player to
their number, contacts, messages and photos. Never return a session value: if it changes, the
player gets an empty phone and the old rows are orphaned.
{% endhint %}

{% hint style="warning" %}
`addBank` and `removeBank` must return an **honest boolean**. The phone debits the sender
before crediting the recipient and refunds when the credit fails. Returning `true` when
nothing happened makes money disappear.
{% endhint %}

### Inventory functions (`NashInvCustom`)

Independent from the framework: you can keep ESX and replace only the inventory.

| Function | Returns |
| -------- | ------- |
| `hasItem(src, item)` | Boolean: does this player own the item |
| `itemExists(name)` | `true` it exists, `false` it does not (the phone warns the admin), `nil` you cannot tell (the phone stays silent) |

The third answer of `itemExists` matters. Returning `nil` is not a failure: it stops the phone
from sending an administrator after a problem that may not exist.

## Checking what was picked

With `Config.Debug = true` in [config/main.lua](config-main.md):

| Command | What it shows |
| ------- | ------------- |
| `/phonedeps` | Dependency status and what stops working without each one |
| `/phoneconfig` | Summary of the configuration the server actually read |
| `/phonecheck` | Short verdict: what is preventing the phone from working |

`/phoneconfig` also reports whether the phone item is declared in the detected inventory, which
is the usual reason a phone cannot be opened at all.
