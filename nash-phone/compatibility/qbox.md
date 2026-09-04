# QBOX

QBOX is a QBCore derivative, but it is not used like one. Its accesses go through **exports**
(`exports.qbx_core:GetPlayer`) rather than a central core object fetched once, and it leans on
`ox_lib` and `ox_inventory`: both of which NASH Phone already uses. It therefore gets its own
bridge file rather than reusing the QBCore one.

## Detection

QBOX is picked when `GetResourceState('qbx_core') == 'started'`, and it is tested **first**,
before QBCore and ESX. To skip detection:

```lua
-- config/bridge.lua
Config.Framework = 'qbox'
```

{% hint style="info" %}
The order is not a detail. `qbx_core` frequently runs alongside a `qb-core` compatibility
resource; checking QBCore first would file a QBOX server under the wrong bridge and make it
miss `exports.qbx_core:*` entirely. On a hybrid install, QBOX wins.
{% endhint %}

Player loading is picked up through the `QBCore:Server:OnPlayerLoaded` event, which QBOX also
fires.

## What the bridge maps

| Interface | QBOX implementation |
| --------- | ------------------- |
| `Framework.getPlayer(src)` | `exports.qbx_core:GetPlayer(src)`, inside a `pcall` |
| `Framework.identifier(src)` | `PlayerData.citizenid` |
| `Framework.group(src)` | `exports.qbx_core:HasPrimaryGroup(src, 'admin')`, then `lib.getPlayerGroup(src)` |
| `Framework.fullName(src)` | `PlayerData.charinfo.firstname .. ' ' .. charinfo.lastname` |
| `Framework.getBank(src)` | `PlayerData.money.bank` |
| `Framework.getCash(src)` | `PlayerData.money.cash` |
| `Framework.addBank(src, amount)` | `Player.Functions.AddMoney('bank', amount, 'nash_phone')` |
| `Framework.removeBank(src, amount)` | `Player.Functions.RemoveMoney('bank', amount, 'nash_phone')` |
| `Framework.players()` | `exports.qbx_core:GetQBPlayers()` |
| `Framework.usableItem(name, cb)` | `exports.ox_inventory:RegisterUsableItem(name, cb)` |
| `Framework.job(src)` | `PlayerData.job`, flattened like QBCore |
| Native inventory | **Not implemented:** returns `nil` on purpose |

Every export call is wrapped in a `pcall`, so a QBOX version that renamed one of them degrades
to the neutral value instead of throwing.

## Identity is the citizenid

Same rule as QBCore: the `owner` column of every phone table stores `PlayerData.citizenid`, so
the phone follows the **character**, not the account. Each life has its own number, contacts
and messages.

## Groups

QBOX delegates permissions to `ox_lib`. The bridge tries the dedicated export first
(`HasPrimaryGroup(src, 'admin')`, which yields the group name `'admin'` when true), then falls
back to `lib.getPlayerGroup(src)`. It returns `nil` when neither answers.

Groups gate the `/phone*` test commands through `Config.DebugGroups`, default
`{ 'admin', 'superadmin' }`. On a QBOX server using different ox_lib group names, add them to
that list or use the `nash_phone.debug` ACE.

## Jobs

Identical shape to QBCore, including the real duty state:

```lua
{ name = 'police', grade = 3, gradeName = 'sergeant', label = 'Police', onDuty = true }
```

`onDuty` comes from `PlayerData.job.onduty`, so `Config.Services.ownDuty = false` in
`config/services.lua` is a viable choice if you already have a locker or a clock-in system
driving `exports.nash_phone:SetServiceDuty(source, true)`. With the shipped default
(`ownDuty = true`) the phone owns the Duty switch itself.

## The item, and why `ox_inventory` is effectively required

QBOX does not register usable items in the core: `ox_inventory` does. The bridge therefore
routes `Framework.usableItem` to `exports.ox_inventory:RegisterUsableItem`, and **skips the
call entirely** when `ox_inventory` is not started.

Consequences on a QBOX server without `ox_inventory`:

- using the phone item from the inventory does nothing;
- the phone still opens with `/phone` and the open key, provided the player carries the item;
- `Config.Inventory = 'framework'` is a dead end, because the QBOX bridge deliberately returns
  `nil` for both native-inventory functions. `Inventory.hasItem` only accepts a literal `true`,
  so with `Config.UseItem = true` **nobody** would be able to open the phone.

{% hint style="warning" %}
On QBOX, leave `Config.Inventory` on `'auto'` (which picks `ox_inventory`) or set it to
`'ox'`. Do not force `'framework'`.
{% endhint %}

## Manual steps

### 1. Create the phone item

Declare `Config.ItemName` (default `'phone'`) in `ox_inventory/data/items.lua`:

```lua
['phone'] = {
    label = 'Phone',
    weight = 190,
    stack = false,
    close = true,
    description = 'A smartphone.',
    client = { image = 'phone.png' },
},
```

The SQL helpers in `sql/` target the default ESX `items` table and are not for QBOX.

### 2. Create the earbuds item (optional)

Same file, using `Config.Earbuds.itemName` (default `'phone_earbuds'`).

### 3. Declare your jobs in the Services app

`config/services.lua` ships with `police`, `ambulance`, `mechanic` and `taxi`. The `job` field
must match your QBOX job name character for character.

## Test status

{% hint style="warning" %}
The QBOX bridge is complete and every one of its calls is guarded, but it has not accumulated
the production hours that the ESX path has, and the shipped SQL helpers assume ESX. Run the
checklist below before opening a QBOX server.
{% endhint %}

## Pitfalls

- **Detected as `qbcore` instead of `qbox`.** That means `qbx_core` was not `started` when the
  phone resolved the bridge. Check `server.cfg` order, or force `Config.Framework = 'qbox'`.
- **`Config.Inventory = 'framework'` breaks item ownership.** See above: QBOX has no native
  inventory to read.
- **`charinfo` must be a table.** `Framework.fullName` returns `nil` otherwise, and the
  translated "Unknown" fallback shows everywhere.
- **Changing framework later orphans everything.** Phone tables are keyed on the citizenid.
  See the warning in the [Compatibility Overview](README.md).

## Checklist

1. `[nash_phone] framework détecté : qbox.` appears in the console at start: not `qbcore`.
2. `/phonedeps` reports `qbx_core` as the framework and `ox_inventory` as the inventory.
3. `/phonedeps` reports the phone item as declared.
4. Using the item from `ox_inventory` opens the phone; so do `/phone` and the open key; and
   both refuse a player who does not carry the item.
5. Settings shows the character's real RP name, not the translated "Unknown".
6. The Bank app shows the same balance as `PlayerData.money.bank`, and a transfer between two
   connected players moves money on both sides.
7. A player with the `police` job sees their service page in the Services app.
8. `/phonehelp` is accepted for an admin and refused for a plain player.
