# QBCore

## Detection

QBCore is picked when `GetResourceState('qb-core') == 'started'`, after QBOX has been ruled
out. To skip detection:

```lua
-- config/bridge.lua
Config.Framework = 'qbcore'
```

The core object is fetched at the **first use of the bridge**, not at resource start:

```lua
local QB = exports['qb-core']:GetCoreObject()
```

Calling `GetCoreObject()` at load time on a server without `qb-core` would throw before the
config had even been read, which is why no provider file calls anything while loading.

Player loading is picked up through the `QBCore:Server:OnPlayerLoaded` event.

{% hint style="warning" %}
If you run `qbx_core` **and** a `qb-core` compatibility resource side by side, auto-detection
picks **QBOX**, not QBCore: `qbox` is tested first on purpose. See [QBOX](qbox.md).
{% endhint %}

## What the bridge maps

| Interface | QBCore implementation |
| --------- | --------------------- |
| `Framework.getPlayer(src)` | `QB.Functions.GetPlayer(src)` |
| `Framework.identifier(src)` | `PlayerData.citizenid` |
| `Framework.group(src)` | `QB.Functions.GetPermission(src)` |
| `Framework.fullName(src)` | `PlayerData.charinfo.firstname .. ' ' .. charinfo.lastname` |
| `Framework.getBank(src)` | `PlayerData.money.bank` |
| `Framework.getCash(src)` | `PlayerData.money.cash` |
| `Framework.addBank(src, amount)` | `Player.Functions.AddMoney('bank', amount, 'nash_phone')` |
| `Framework.removeBank(src, amount)` | `Player.Functions.RemoveMoney('bank', amount, 'nash_phone')` |
| `Framework.players()` | `QB.Functions.GetQBPlayers()` |
| `Framework.usableItem(name, cb)` | `QB.Functions.CreateUseableItem(name, cb)` |
| `Framework.job(src)` | `PlayerData.job`, flattened |
| Native inventory: item held | `Player.Functions.GetItemByName(item).amount > 0` |
| Native inventory: item list | `QB.Shared.Items` |

Money reasons are logged as `nash_phone` on both sides.

## Identity is the citizenid, not the license

The `owner` column of every phone table stores `PlayerData.citizenid` (for example `JSA12345`).

That is a deliberate choice with a visible consequence: QBCore hands out a citizenid **per
character**, so the phone follows the character, exactly as it does on a multicharacter ESX
server. Each life has its own number, contacts and messages. Using the license instead would
have given one shared phone across all of a player's characters and leaked one life's texts
into another.

## Groups

`QB.Functions.GetPermission(src)` is called inside a `pcall`, and both return shapes are
handled:

- a **string:** returned as-is;
- a **table** such as `{ god = true, admin = true }`: scanned for `god`, `superadmin`,
  `admin`, `mod`, in that order, and the first match is returned as the group name.

The bridge returns `nil`, never `'user'`, when it cannot tell: refusing an admin who has just
connected is worse than returning nothing.

Groups gate the `/phone*` test commands through `Config.DebugGroups`, default
`{ 'admin', 'superadmin' }`. On a QBCore server whose permission names differ, either add them
to that list or use the `nash_phone.debug` ACE instead.

## Money

`AddMoney` and `RemoveMoney` return a boolean in QBCore, and the bridge passes it straight
through (`~= false`). QBCore refuses an overdrawn withdrawal itself and returns `false`, so the
bridge does **not** read the balance first: doing so would open a race between the read and
the withdrawal.

The booleans matter: a phone transfer debits the sender, credits the recipient, and refunds
the sender when the credit fails.

## Jobs

QBCore stores the grade in a `grade = { level, name }` sub-table where ESX stores a flat
number. The bridge flattens both into one shape, so no app ever has to know which framework is
running:

```lua
{ name = 'police', grade = 3, gradeName = 'sergeant', label = 'Police', onDuty = true }
```

`onDuty` is read from `PlayerData.job.onduty`. QBCore has a real duty state, unlike ESX, which
means you have a choice in `config/services.lua`:

| `Config.Services.ownDuty` | Behaviour |
| ------------------------- | --------- |
| `true` (shipped default) | The phone owns the Duty switch in the Services app and remembers it across reconnections |
| `false` | The switch disappears and the phone waits for another resource to call `exports.nash_phone:SetServiceDuty(source, true)` |

{% hint style="warning" %}
`ownDuty = false` with nothing calling the export leaves **every** service showing
"Unavailable" forever. That is not a bug: no employee has ever been declared on duty.
{% endhint %}

## Manual steps

### 1. Create the phone item

The resource creates no item. Declare `Config.ItemName` (default `'phone'`) in
`qb-core/shared/items.lua`, using the same name as the config key. The SQL helpers in `sql/`
target the ESX `items` table and are **not** for QBCore.

If your server runs `ox_inventory` on top of QBCore, declare the item in
`ox_inventory/data/items.lua` instead: `ox_inventory` holds the items in that case, and the
inventory bridge detects it first.

### 2. Create the earbuds item (optional)

Same file, using `Config.Earbuds.itemName` (default `'phone_earbuds'`). Without it, private
listening is unreachable and the console says so in yellow; nothing else breaks.

### 3. Declare your jobs in the Services app

`config/services.lua` ships with `police`, `ambulance`, `mechanic` and `taxi`. The `job` field
must match your QBCore job name character for character: the comparison is strict, with no
case folding and no trimming.

## Test status

{% hint style="warning" %}
The QBCore bridge is complete and uses only documented `QBCore.Functions.*` and `PlayerData`
paths, which makes it resilient to minor version bumps. It has not accumulated the production
hours that the ESX path has, and the shipped SQL helpers assume ESX. Treat the checklist below
as required, not optional, before opening a QBCore server.
{% endhint %}

## Pitfalls

- **The item gate has no back door.** `/phone` and the open key both go through the same
  server check. If the item is not declared, nobody opens the phone: not even an admin.
  Fifteen seconds after start the console prints a red block saying exactly that.
- **`charinfo` must be a table.** `Framework.fullName` reads `charinfo.firstname` and
  `charinfo.lastname` directly and returns `nil` when `charinfo` is not a table. Forks that
  store it as a JSON string will show the translated "Unknown" fallback everywhere; that needs
  a custom bridge.
- **Non-standard permission names.** `GetPermission` returning something other than a string
  or a `god` / `superadmin` / `admin` / `mod` table yields `nil`, and the `/phone*` test
  commands then only accept the `nash_phone.debug` ACE.
- **Changing framework later orphans everything.** Phone tables are keyed on the citizenid.
  See the warning in the [Compatibility Overview](README.md).

## Checklist

1. `[nash_phone] framework détecté : qbcore.` appears in the console at start: **not**
   `qbox`, unless you meant it.
2. `/phonedeps` reports `qb-core` as the framework in use, and the expected inventory.
3. `/phonedeps` reports the phone item as declared.
4. The phone opens with the item in the player's pocket, and refuses without it.
5. Settings shows the character's real RP name, not the translated "Unknown".
6. The Bank app shows the same balance as `PlayerData.money.bank`, and a transfer between two
   connected players moves money on both sides: with no money created when the recipient
   disconnects mid-transfer.
7. A player with the `police` job sees their service page in the Services app, and the Duty
   switch is present (`ownDuty = true`) or absent (`ownDuty = false`) as configured.
8. `/phonehelp` is accepted for an admin and refused for a plain player.
