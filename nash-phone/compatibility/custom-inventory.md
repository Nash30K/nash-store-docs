# Custom Inventory

The inventory bridge is **independent** of the framework bridge. You can keep ESX, QBCore or
QBOX and replace only the inventory: which is the normal case, since an ESX server running
`ox_inventory` is the most common setup there is.

## What the phone asks of an inventory

Four questions. Nothing else. There is no add and no remove: the phone creates no item and
never moves one.

| Question | Interface | Used for |
| -------- | --------- | -------- |
| Does this player carry this item? | `Inventory.hasItem(src, item)` | Opening the phone (`Config.UseItem`), pairing the earbuds (`Config.Earbuds.requireItem`) |
| Is this item **declared** on the server? | `Inventory.itemExists(name)` | The startup warning, and `/phonedeps` |
| What is the serial of the device this player carries? | `Inventory.serial(src, item)` | Recognising a **new phone**, which asks for the first-run setup again, and **Face ID**, which recognises one character per phone |
| Does this player still carry the device with this serial? | `Inventory.porteSerial(src, item, serial)` | The same, asked in the safer direction |

The last two are **optional**. They only work on inventories that keep data **per item
instance**, and they return `nil` everywhere else, in which case the phone behaves exactly as it
did before this feature existed: the setup never replays, and Face ID recognises everybody. See
[A new phone asks for the setup again](#a-new-phone-asks-for-the-setup-again) below, and
[Settings > Face ID](../apps/README.md#system).

## Built-in providers

| Mode | Detected on | How it answers `hasItem` | How it answers `itemExists` |
| ---- | ----------- | ------------------------ | --------------------------- |
| `ox` | `ox_inventory` started | `exports.ox_inventory:Search(src, 'count', item) > 0` | `exports.ox_inventory:Items()` |
| `qb` | `qb-inventory` **or** `lj-inventory` started | The framework player object's `Functions.GetItemByName(item).amount > 0` | The framework's item table |
| `framework` | Always accepts: last resort | The framework bridge's `invHasItem` | The framework bridge's `invItems` |
| `custom` | Never: you name it | `NashInvCustom.hasItem` | `NashInvCustom.itemExists` |

Device recognition, provider by provider:

| Mode | Recognises a new phone? | How |
| ---- | ----------------------- | --- |
| `ox` | Yes, fully | The serial lives in the item's `metadata.serial`. `GetSlotWithItem` is asked for **that exact serial**, so carrying two phones at once changes nothing |
| `qb` | Best effort | The serial lives in the item's `info.serial`, written through the core's `SetItemData`. That function is checked before being called: on a fork that does not have it, nothing is written and recognition stays off. A new serial is written on the **first** matching item; checking a known serial reads **every** copy in `PlayerData.items`, so carrying two phones does not fail Face ID |
| `framework` | Only if you provide it | The framework bridge's `invSerial` and `invPorteSerial`, both shipped returning `nil` |
| `custom` | Only if you provide it | `NashInvCustom.serial` and `NashInvCustom.porteSerial`, both shipped returning `nil` |

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

## The functions to implement

At the bottom of `server/bridge/custom.lua`, in the global `NashInvCustom` table. All of them
run on the server. The first two are required; the last two are optional and ship returning
`nil`.

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

<details>

<summary>serial(src, item) : optional</summary>

Returns the serial of the device this player is carrying, or `nil`.

The serial must live **inside the item**, not in your database and not on the player. Write it
the first time you read one, then return it unchanged from then on. 64 characters maximum: a
longer string is refused, because the column that stores it would truncate it and the phone
would see a different serial every time.

```lua
serial = function(src, item)
    local slot = exports['your-inventory']:GetFirstSlot(src, item)
    if not slot then return nil end                 -- not carrying one: say nothing
    if slot.metadata and slot.metadata.serial then return slot.metadata.serial end

    local fresh = ('%x-%08x'):format(os.time(), math.random(0, 0x7FFFFFFF))
    exports['your-inventory']:SetMetadata(src, slot.id, { serial = fresh })
    return fresh
end,
```

{% hint style="danger" %}
**Never derive the serial from the player.** Their identifier, their phone number, their
character name: all of these are identical on every device they will ever hold, so a brand new
phone would reopen the previous one's session and the whole feature would do nothing. The value
has to belong to the object.
{% endhint %}

{% hint style="warning" %}
If your inventory merges identical items into one stacked line with a quantity, there is nothing
to distinguish and you should leave this returning `nil`. That is a deliberate, supported answer,
not a gap: it keeps the behaviour the phone had before this feature.
{% endhint %}

</details>

<details>

<summary>porteSerial(src, item, serial) : optional</summary>

Answers whether the player **still** carries the device with that serial.

| Return | Meaning | What the phone does |
| ------ | ------- | ------------------- |
| `true` | Yes, this is their usual phone | Nothing |
| `false` | No | Reads the current serial, and asks for the setup again if it is a different one |
| `nil` | You cannot tell | Nothing for the setup. Face ID falls back on `serial`: the player is recognised only if the serial it returns is theirs |

If you implement `serial`, implement `porteSerial` too. With `serial` alone, Face ID can only
compare the **first** phone your inventory returns: a player who carries somebody else's phone
before their own would be refused on it.

```lua
porteSerial = function(src, item, serial)
    return exports['your-inventory']:HasItemWithMetadata(src, item, { serial = serial })
end,
```

The question is asked in this direction on purpose. Asking "which serial does this player's
phone have" forces your inventory to pick one when the player carries two, and nothing
guarantees it picks the same one twice: the setup would replay every other time the phone
opened.

</details>

## A new phone asks for the setup again

Every device gets a serial stamped into its item the first time the phone reads one. When the
player opens a phone whose serial does not match the one recorded for their character, the
first-run setup runs again, and the byCloud account is reopened with its password: exactly like
a real handset.

- Putting the phone in a trunk and taking it back changes nothing. The serial is the same.
- **Only the setup row is reset.** Messages, contacts, the camera roll and the phone number all
  belong to the character and to their line, not to the handset. A player who loses their phone
  does not lose their texts.
- **The byCloud account is untouched.** It never belonged to the device, which is why it can be
  reopened from the new one with its password.
- **Phones already in circulation keep their setup.** The first serial read is adopted without
  erasing anything; only the *next* device counts as new.
- With `Config.UseItem = false` there is no item, therefore no device to recognise, and the
  setup never replays.

`/phonesetup` prints the recorded serial next to the one currently in hand, which is the fastest
way to tell "different handset" from "setup was never saved".

A resource that wants to know fires on `nash-phone:deviceChanged`: see
[Server Events](../developer-api/server-events.md).

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
- **Returning something that is neither a boolean nor a count.** `hasItem` accepts `true`, a
  number greater than zero, and the string `'1'`, which covers every sane way of writing one.
  It does **not** accept a table or an arbitrary string: Lua would call those true, and a
  malformed return would pass for ownership.
- **Deriving the serial from the player instead of the item.** Recognition then never fires: a
  new handset carries the same value as the old one.
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
8. Only if you implemented `serial`: `/phonesetup` shows the same value on both lines, opening
   and closing the phone several times never changes it, and storing the phone in a trunk and
   taking it back does not change it either.
9. Only if you implemented `serial`: destroying the phone item, receiving a fresh one and
   opening it runs the first-run setup again, and the byCloud account reopens with its
   password. The player's texts and contacts are still there afterwards.
