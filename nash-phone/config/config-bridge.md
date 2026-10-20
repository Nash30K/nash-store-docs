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
| `'esx'` | ESX Legacy (older ESX 1.x also works, through the `esx:getSharedObject` event) |
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
| `'qs'` | `qs-inventory` |
| `'qb'` | `qb-inventory`, `ps-inventory` and `lj-inventory` |
| `'framework'` | The framework's own inventory (ESX, QBCore) |
| `'custom'` | Your own, written in `server/bridge/custom.lua` |

Automatic detection tries **ox, then qs, then qb, then framework**. `ox_inventory` comes first
because when it is running it is the resource that actually holds the items, whatever framework
sits underneath. `qs-inventory` comes before `qb` because on QBCore it replaces `qb-inventory`.

The phone asks an inventory three things: does this player own an item (the phone, and the
earbuds of `Config.Earbuds`), is that item declared on the server, and which serial the phone
item carries, so that a different handset replays the first-run setup (see
[A new phone asks again](config-setup.md#a-new-phone-asks-again)). The item settings are
`Config.UseItem` / `Config.ItemName` in [config/main.lua](config-main.md). qs-inventory also
registers the usable item itself.

An unknown value prints `Config.Inventory = "..." : inconnu. Attendu : auto, ox, qs, qb,
framework, custom.` in the server console, and the phone then sees no inventory at all.

## Custom bridge

Set `Config.Framework = 'custom'` and/or `Config.Inventory = 'custom'`, then fill
`server/bridge/custom.lua` and restart the resource. Apart from the files of `config/`, it is the
only Lua file of the phone you can read and edit: every other Lua file is protected by FiveM
Asset Escrow.

`server/bridge/custom.lua` never self-detects. It is used only when you name it here.

### Framework functions (`NashFw.custom`)

Only `identifier` is mandatory. The template's functions already return neutral values
(`nil`, `0`, `false`), so the phone boots before you have filled anything. A function you delete
from the table is replaced by a neutral value and writes one console warning, so you can wire
things up in stages.

| Function | Required | What you lose if left empty |
| -------- | -------- | --------------------------- |
| `identifier(src)` | **Yes** | Nothing works. This is the key of every phone table: a string of 64 characters at most |
| `fullName(src)` | No | The RP name shows as "Unknown" (translated) everywhere |
| `getPlayer(src)` | No | Only passed back to other functions of the same bridge |
| `group(src)` | No | Nothing: the phone does not read it at the moment |
| `job(src)` | No | The Services app recognises no employee |
| `getBank(src)` / `getCash(src)` | No | The Wallet app shows 0 |
| `addBank(src, amount)` / `removeBank(src, amount)` | No | Transfers fail |
| `players()` | No | Returns an empty list |
| `usableItem(name, cb)` | No | The phone no longer opens from the item, only from the key |
| `setPhoneNumber(src, number)` | No | `Config.Phone.syncFramework` copies nothing into the character data |
| `invHasItem(src, item)` | No | Only used when `Config.Inventory = 'framework'`: nobody holds the item |
| `invItems()` | No | Only used with `'framework'` (and `'qb'`): the item-declaration check cannot run |
| `invSerial(src, item)` | No | Only used with `'framework'`: a new handset is not recognised, the setup never replays |
| `invPorteSerial(src, item, serial)` | No | Only used with `'framework'`: same as above |

`invSerial` returns the serial stored **in** the phone item (write one on first read, 64
characters at most), and `invPorteSerial` answers whether the player still carries the item with
that serial: `true`, `false`, or `nil` when you cannot tell. See
[Custom Inventory](../compatibility/custom-inventory.md).

`job(src)` must return exactly this shape, or `nil`:

```lua
{ name = 'police', grade = 3, gradeName = 'sergeant', label = 'Police', onDuty = true }
```

`name` must match the `job` of an entry of [config/services.lua](config-services.md) for the
Services app to recognise the employee. `onDuty` is accepted but not read by the phone: with
`Config.Services.ownDuty = false`, duty only comes from the `SetServiceDuty` export.

`NashFw.custom.playerLoadedEvent` names the event your framework fires when a character is
loaded. Leave it `nil` if you have none: the phone catches up on the first NUI start instead,
a little later.

`NashFw.custom.playerUnloadedEvent` names the server event fired when a player goes back to
character selection without disconnecting, with the server ID as first argument. The phone
then cleans up as on a disconnection: it ends the call in progress, closes the byCloud session,
stops the music, takes the player off duty in Services and ends their live stream. On the
player's client, the phone closes and saves nothing until the next character is loaded. Leave
it `nil` without multiple characters.

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
| `serial(src, item)` | Optional. The serial stored **in** the phone item, written on first read (64 characters at most), or `nil` if your inventory keeps no data per item. Without it a new handset never replays the setup |
| `porteSerial(src, item, serial)` | Optional. Does the player still carry the item with this serial: `true`, `false`, or `nil` when unknown |
| `usableItem(name, cb)` | Optional, not in the template. Register the usable item yourself and return `true`, so the framework does not |

Never derive the serial from the player (identifier, number): it must live in the item, or every
phone of a player would look like the same handset. See
[Custom Inventory](../compatibility/custom-inventory.md).

The third answer of `itemExists` matters. Returning `nil` is not a failure: it stops the phone
from sending an administrator after a problem that may not exist.

## Checking what was picked

Read the server console at startup:

| Line | Meaning |
| ---- | ------- |
| `[nash_phone] framework détecté : <name>.` / `[nash_phone] inventaire détecté : <name>.` | Automatic detection found that bridge |
| `[nash_phone] framework : <name> (imposé par la configuration).` / `[nash_phone] inventaire : <name> (imposé par la configuration).` | The bridge was forced in this file |
| `[nash_phone] inventaire : pont personnalisé.` | `Config.Inventory = 'custom'` and `NashInvCustom` was found |
| `[nash_phone] AUCUN framework détecté pour l'instant. Le téléphone ne pourra identifier personne.` | No framework found: start your core **before** `nash_phone`, or set `'custom'` |
| `[nash_phone] Config.Inventory = "custom" mais server/bridge/custom.lua ne définit pas NashInvCustom.` | `'custom'` is set but the inventory table of `server/bridge/custom.lua` is missing |

About fifteen seconds after startup, a red block starting with
`[nash_phone] Config.UseItem = true but the item "phone" DOES NOT EXIST` (with your
`Config.ItemName`) means the detected inventory does not know the phone item, which is the usual
reason a phone cannot be opened at all. See [Phone item](config-main.md#phone-item). When the
inventory cannot tell (an `itemExists` that returns `nil`), the block is not printed.
