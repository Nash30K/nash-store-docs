# ESX

ESX is the reference platform for NASH Phone. It is the framework listed in the resource's
required dependencies, and the two SQL helpers shipped in `sql/` target the default ESX
`items` table.

## Detection

ESX is picked when `GetResourceState('es_extended') == 'started'`, after QBOX and QBCore have
been ruled out. To skip detection:

```lua
-- config/bridge.lua
Config.Framework = 'esx'
```

The core object is fetched at the **first use of the bridge**, not at resource start:

```lua
local ESX = exports['es_extended']:getSharedObject()
```

That matters for start order: the file is loaded on every server, but the export is only
called once something actually needs a player, which is long after `es_extended` has booted.

Player loading is picked up through the `esx:playerLoaded` event.

## What the bridge maps

| Interface | ESX implementation |
| --------- | ------------------ |
| `Framework.getPlayer(src)` | `ESX.GetPlayerFromId(src)` |
| `Framework.identifier(src)` | `xPlayer.identifier` |
| `Framework.group(src)` | `xPlayer.group`, then `xPlayer.getGroup()` |
| `Framework.fullName(src)` | `xPlayer.getName()`, then `get('firstName') .. ' ' .. get('lastName')`, then `xPlayer.name` |
| `Framework.getBank(src)` | `xPlayer.getAccount('bank').money` |
| `Framework.getCash(src)` | `xPlayer.getAccount('money').money` |
| `Framework.addBank(src, amount)` | `xPlayer.addAccountMoney('bank', amount)` |
| `Framework.removeBank(src, amount)` | Balance check, then `xPlayer.removeAccountMoney('bank', amount)` |
| `Framework.players()` | `ESX.GetExtendedPlayers()` |
| `Framework.usableItem(name, cb)` | `ESX.RegisterUsableItem(name, cb)` |
| `Framework.job(src)` | `xPlayer.job`, or `xPlayer.getJob()` |
| Native inventory: item held | `xPlayer.getInventoryItem(item).count > 0` |
| Native inventory: item list | `ESX.Items` |

The last two rows are only read when `Config.Inventory` resolves to `'framework'`.

## Identifier

The phone stores `xPlayer.identifier` (`license:abc123…`) in the `owner` column of every one
of its tables, a `VARCHAR(64)`. On a multicharacter ESX setup, ESX itself hands out a
per-character identifier, so each character gets its own number, contacts and messages.

## Names

`Framework.fullName` returns `nil`, never a literal `"Unknown"`, when ESX does not know the
player yet. That is deliberate: the fallback text is shown to a reader whose phone may be in
another language, so each caller translates `unknown_name` for its own recipient.

An **empty** name counts as an absent name: without that check, the caller would receive an
empty string and print a hole where its translated fallback belonged.

## Groups

Both spellings are tried, because both exist depending on the ESX version: the `group` field
on ESX Legacy, the `getGroup()` method on older builds. `getGroup` is called **without**
`self`, since ESX attaches it as a closure on the player table rather than as a method.

The bridge returns `nil`, never `'user'`, when the framework does not know the player yet -
confusing "not loaded" with "plain player" would reject an admin who has just connected.

Groups are used by `Config.DebugGroups` (default `{ 'admin', 'superadmin' }`) to gate the
`/phone*` test commands.

## Money

`Framework.removeBank` reads the account and only calls `removeAccountMoney` when the balance
covers the amount, then returns `true`. `Framework.addBank` returns `true` once
`addAccountMoney` has run.

Those booleans are load-bearing. A phone transfer debits the sender, then credits the
recipient, then refunds the sender if the credit failed. A bridge that returns `true` when
nothing moved makes money disappear.

## Jobs

`Framework.job` normalises ESX's job table into the shape every app expects:

```lua
{ name = 'police', grade = 3, gradeName = 'sergeant', label = 'Police' }
```

It returns `nil`: not an empty table: while the player is not loaded. "Not known yet" is not
"unemployed".

{% hint style="warning" %}
**ESX has no duty state**, so the bridge does not fill `onDuty`. Leave
`Config.Services.ownDuty = true` (the shipped default): the phone then owns the Duty switch in
the Services app and remembers it across reconnections. Setting it to `false` on ESX hides the
switch and leaves every service permanently "Unavailable" until another resource calls
`exports.nash_phone:SetServiceDuty(source, true)`.
{% endhint %}

## Manual steps

Auto-detection covers the framework itself. These are on you.

### 1. Create the phone item

The resource creates **no** item. With `Config.UseItem = true` (the default), the item named
in `Config.ItemName` (default `'phone'`) must exist in your inventory **and** be in the
player's pockets.

For the default ESX inventory: the one that reads its items from the `items` table: import
`sql/item_phone.sql`, or run:

```sql
INSERT INTO `items` (`name`, `label`, `weight`) VALUES ('phone', 'Phone', 1)
  ON DUPLICATE KEY UPDATE `label` = `label`;
```

Then **restart `es_extended`**: ESX only re-reads the `items` table at startup.

If you run `ox_inventory` on top of ESX: the most common setup: that table is not used.
Declare the item in `ox_inventory/data/items.lua` instead:

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

`ox_inventory` forwards item use to `ESX.RegisterUsableItem`, which the phone already
registers, so there is nothing else to wire.

### 2. Create the earbuds item (optional)

`Config.Earbuds` is enabled by default and expects `Config.Earbuds.itemName`
(default `'phone_earbuds'`). Import `sql/item_earbuds.sql` for the default ESX inventory, or
declare it in `ox_inventory/data/items.lua`. Missing, the phone still runs: only private
listening is unreachable, and the console says so in yellow.

### 3. Declare your jobs in the Services app

`config/services.lua` ships with `police`, `ambulance`, `mechanic` and `taxi`. The `job` field
must match your framework's job name **character for character:** the comparison is strict,
with no case folding and no trimming, precisely so a typo is visible instead of silently
matching nothing.

## Pitfalls

- **The item gate has no back door.** `/phone` and the open key both go through the same
  server check (`server/main.lua`, `requestOpen`). If the item does not exist, nobody opens
  the phone: not even an admin. Fifteen seconds after start the console prints a red block
  saying exactly that. Either create the item or set `Config.UseItem = false`.
- **`xPlayer.get(...)` returns nothing on very old forks.** The name chain falls through to
  `xPlayer.name` and finally to `nil`, and the caller substitutes its own translated fallback.
- **Money accounts are named `bank` and `money`.** ESX's cash account is `money`, not `cash`.
  A fork that renamed either account needs a custom bridge.
- **Changing framework later orphans everything.** See the warning in the
  [Compatibility Overview](README.md).

## Checklist

1. `[nash_phone] framework détecté : esx.` appears in the console at start.
2. `/phonedeps` reports `es_extended` as the framework in use, and the expected inventory.
3. `/phonedeps` reports the phone item as declared.
4. The phone opens with the item in the player's pocket, and refuses without it.
5. Settings shows the character's real RP name, not the translated "Unknown".
6. The Bank app shows the same balance as the framework, and a transfer between two connected
   players moves money on both sides.
7. A player with the `police` job sees the "My service" page in the Services app.
