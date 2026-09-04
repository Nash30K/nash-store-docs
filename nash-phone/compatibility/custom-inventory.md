# Custom Inventory

The inventory bridge is **independent** of the framework bridge. You can keep ESX, QBCore or
QBOX and replace only the inventory: which is the normal case, since an ESX server running
`ox_inventory` is the most common setup there is.

## What the phone asks of an inventory

Two questions. Nothing else. There is no add, no remove, no metadata, no slot.

| Question | Interface | Used for |
| -------- | --------- | -------- |
| Does this player carry this item? | `Inventory.hasItem(src, item)` | Opening the phone (`Config.UseItem`), pairing the earbuds (`Config.Earbuds.requireItem`) |
| Is this item **declared** on the server? | `Inventory.itemExists(name)` | The startup warning, and `/phonedeps` |

The phone creates no item and never moves one.

## Built-in providers

| Mode | Detected on | How it answers `hasItem` | How it answers `itemExists` |
| ---- | ----------- | ------------------------ | --------------------------- |
| `ox` | `ox_inventory` started | `exports.ox_inventory:Search(src, 'count', item) > 0` | `exports.ox_inventory:Items()` |
| `qb` | `qb-inventory` **or** `lj-inventory` started | The framework player object's `Functions.GetItemByName(item).amount > 0` | The framework's item table |
| `framework` | Always accepts: last resort | The framework bridge's `invHasItem` | The framework bridge's `invItems` |
| `custom` | Never: you name it | `NashInvCustom.hasItem` | `NashInvCustom.itemExists` |

Auto order is `ox`, then `qb`, then `framework`. The full detection rules are in the
[Compatibility Overview](README.md).

{% hint style="info" %}
The `qb` provider goes through the **framework player object** rather than an inventory export,
because qb-inventory forks rename their exports often while `GetItemByName` stays stable on the
core. The consequence: it needs a framework whose player object exposes
`Functions.GetItemByName`. QBOX does not: on QBOX, use `ox_inventory`.
{% endhint %}

## Inventories that are not auto-detected

`qs-inventory` is not referenced anywhere in the resource. Neither is any other inventory
outside the table above. With `Config.Inventory = 'auto'`, such a server silently falls through
to the `framework` provider, which will answer from the framework's native inventory: usually
the wrong source once a third-party inventory has taken over the items.

Symptom: players carry the phone item, but the phone refuses to open.

Fix: fill in the custom bridge below.

## Enabling custom mode

```lua
-- config/bridge.lua
Config.Inventory = 'custom'
```

```
[nash_phone] inventaire : pont personnalisé.
```

Accepted values: `'auto'` (default), `'ox'`, `'qb'`, `'framework'`, `'custom'`. Anything else
prints:

```
[nash_phone] Config.Inventory = "qs" : inconnu. Attendu : auto, ox, qb, framework, custom.
```

## The two functions to implement

At the bottom of `server/bridge/custom.lua`, in the global `NashInvCustom` table. Both run on
the server.

<details>

<summary>hasItem(src, item)</summary>

Returns `true` when the player carries at least one of `item`.

```lua
hasItem = function(src, item)
    return exports['your-inventory']:GetItemCount(src, item) > 0
end,
```

Only a literal `true` counts. The call is wrapped in a `pcall`, so an error inside your
function is read as "does not have it" rather than crashing the callback: which also means a
broken implementation looks exactly like an empty pocket. Check the console.

Called from four places: opening the phone, the `HasPhoneItem` export, pairing the earbuds, and
a test command.

</details>

<details>

<summary>itemExists(name)</summary>

Answers whether `name` is **declared** on the server. Three return values, and the third is the
point:

| Return | Meaning | What the phone does |
| ------ | ------- | ------------------- |
| `true` | The item exists | Nothing |
| `false` | The item does not exist | Prints the startup warning |
| `nil` | You cannot tell: not loaded yet, no export | Stays quiet |

```lua
itemExists = function(name)
    local items = exports['your-inventory']:GetItemList()
    if type(items) ~= 'table' then return nil end   -- not loaded: say nothing
    return items[name] ~= nil
end,
```

{% hint style="warning" %}
Never return `false` when you simply do not know. A false negative sends a server owner hunting
for a problem that does not exist. `nil` is a valid, deliberate answer, and the shipped stub
returns exactly that.
{% endhint %}

This function has no effect on gameplay: it only drives the startup diagnostic and the
`/phonedeps` line.

</details>

## The items to declare

The resource creates no item. Two names come from the config:

| Config key | Default | Required? |
| ---------- | ------- | --------- |
| `Config.ItemName` | `'phone'` | Yes, when `Config.UseItem = true` (the shipped default) |
| `Config.Earbuds.itemName` | `'phone_earbuds'` | Only when `Config.Earbuds.enabled = true` and `requireItem = true` (both shipped defaults) |

{% hint style="danger" %}
**The phone item gate has no back door.** `/phone` and the open key both go through the same
server check (`server/main.lua`, `requestOpen`). If the item does not exist in your inventory,
nobody opens the phone: not even an admin, not from the console. Fifteen seconds after start
(to let the inventory finish loading its items), the console prints a red block saying exactly
that, and `/phonecheck` reports it as blocking.

Two ways out: declare the item, or set `Config.UseItem = false` to open the phone without
carrying anything.
{% endhint %}

The earbuds warning is yellow, not red: without that item, private listening is unreachable and
nothing else changes.

Where to declare them depends on your inventory:

- `ox_inventory` → `ox_inventory/data/items.lua`
- `qb-inventory` and derivatives → `qb-core/shared/items.lua`
- The default ESX inventory (items in the database `items` table) → import `sql/item_phone.sql`
  and `sql/item_earbuds.sql`, then restart `es_extended`, which only re-reads that table at
  startup
- Anything else → your inventory's own declaration file

## Pitfalls

- **`NashInvCustom` missing.** With `Config.Inventory = 'custom'` and no such table, the phone
  prints
  `[nash_phone] Config.Inventory = "custom" mais server/bridge/custom.lua ne définit pas NashInvCustom.`
  and ends with no inventory at all: `hasItem` always returns `false`, so nobody opens the
  phone while `Config.UseItem = true`.
- **`Config.Inventory = 'framework'` on QBOX.** QBOX has no native inventory, and its bridge
  returns `nil` for both native-inventory functions on purpose. Item ownership then always
  answers `false`.
- **Returning a count instead of a boolean.** `hasItem` is compared against a literal `true`; a
  number is treated as "does not have it".
- **Editing `server/bridge/custom.lua` and losing it on update.** It is the one file in the
  resource meant to be edited. Keep a copy before replacing the folder.

## Checklist

1. `[nash_phone] inventaire : pont personnalisé.` appears at start: or
   `inventaire détecté : ox.` if you meant to stay on auto.
2. `/phonedeps` reports the inventory in use and the phone item as declared.
3. A player carrying the item opens the phone with `/phone`, the open key, and by using the
   item.
4. A player **without** the item is refused on all three paths.
5. Dropping the item and trying again refuses as well: this is the test that catches an
   implementation reading a cache instead of the live inventory.
6. With `Config.Earbuds` enabled: using the earbuds item pairs them, using it again connects
   and disconnects them, and the item is never consumed.
7. Temporarily setting `Config.ItemName` to a name that does not exist produces the red startup
   block. That proves `itemExists` is answering, not returning `nil` by accident.
