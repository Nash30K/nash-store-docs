# Inventory Items

NASH Phone creates **no item**. Your inventory owns the item list, so the two items below have
to be declared on your side before players can carry them.

| Item | Config key | Default | Required |
|---|---|---|---|
| The phone | `Config.ItemName` | `phone` | Yes, while `Config.UseItem = true` |
| Wireless earbuds | `Config.Earbuds.itemName` | `phone_earbuds` | No, the feature is optional |

Both keys are in `config/main.lua`.

{% hint style="success" %}
**Ready-to-paste files ship in the `INSTALL` folder** of the resource:
`INSTALL/ox_inventory/data/items.md`, `INSTALL/qb-core/shared/items.md`,
`INSTALL/qs-inventory/items.md`, and `INSTALL/sql/` for the default ESX inventory. Copy the
entries into your inventory, translate the labels, restart.
{% endhint %}

{% hint style="danger" %}
With `Config.UseItem = true` (the default), there is **no way around the item**. The `/phone`
command and the open key go through a server-side check (`nash_phone:requestOpen`), and the
client exports `ToggleOpen` and `OpenApp` ask the server the same question (the
`nash_phone:canOpen` callback) before opening. Nobody can carry an item that does not exist,
not even an admin. Either declare the item, or set `Config.UseItem = false`.
{% endhint %}

## No `client.export` to add

Unlike most phone scripts, you do not wire a usable-item export. `nash_phone` registers the
handler itself at boot, once the framework is detected (it waits up to 60 seconds for it):

| Setup | What the phone calls |
|---|---|
| `qs-inventory` (any framework) | `exports['qs-inventory']:CreateUsableItem` |
| Custom inventory whose `usableItem` returns `true` | Your `NashInvCustom.usableItem` |
| Otherwise, on ESX | `ESX.RegisterUsableItem` |
| Otherwise, on QBCore | `QBCore.Functions.CreateUseableItem` |
| Otherwise, on QBOX | `exports.qbx_core:CreateUseableItem` |

The inventory is asked first; the framework only when the inventory does not register the item
itself, so the item is never registered twice. The earbuds item is registered the same way when
`Config.Earbuds.enabled` is `true`. `ox_inventory` has no entry of its own: it uses the usable
items of the framework underneath.

You only have to declare the item. Using it opens the phone.

{% hint style="warning" %}
On **QBOX**, the phone registers the item in `qbx_core`, and it is `ox_inventory` that calls it
when a player uses the item. QBOX has no inventory of its own: on a QBOX server without
`ox_inventory` (or `qs-inventory`), using the item does nothing. The open key and `/phone`
still work for anyone who owns the item.
{% endhint %}

## ox_inventory

Add the entries to `ox_inventory/data/items.lua`:

```lua
['phone'] = {
    label = 'Phone',
    weight = 190,
    stack = false,      -- one phone per slot
    close = true,       -- close the inventory when used
    description = 'A smartphone.',
    client = { image = 'phone.png' },
},

['phone_earbuds'] = {
    label = 'Wireless Earbuds',
    weight = 50,
    stack = false,
    close = true,
},
```

The `label` is what players read in their inventory, so translate it to your server's language.

Then:

```
restart ox_inventory
```

## ESX default inventory

The classic ESX inventory reads its items from the `items` table in the database. Two ready
files ship with the resource:

| File | Item | Label it inserts |
|---|---|---|
| `INSTALL/sql/item_phone.sql` | `phone` | `Téléphone` |
| `INSTALL/sql/item_earbuds.sql` | `phone_earbuds` | `Écouteurs sans fil` |

Both files ship with French labels: on a server in another language, change the label before
importing them. Or run the inserts directly, with English labels:

```sql
INSERT INTO `items` (`name`, `label`, `weight`) VALUES ('phone', 'Phone', 1)
  ON DUPLICATE KEY UPDATE `label` = `label`;

INSERT INTO `items` (`name`, `label`, `weight`) VALUES ('phone_earbuds', 'Wireless Earbuds', 1)
  ON DUPLICATE KEY UPDATE `label` = `label`;
```

Some ESX versions have `rare` and `can_remove` columns as well; both SQL files carry the
alternative statement in a comment.

{% hint style="info" %}
Restart `es_extended` after the import. ESX only reads the `items` table at startup, so a
freshly inserted item stays invisible until it does.
{% endhint %}

## qs-inventory

Detected automatically. Declare the two items in qs-inventory's item list, with the fields its
documentation uses (see Quasar's *Item Configuration* page), and put the images in
`qs-inventory/html/images/`:

```lua
['phone'] = {
    name = 'phone',
    label = 'Phone',
    weight = 190,
    type = 'item',
    image = 'phone.png',
    unique = true,
    useable = true,
    shouldClose = true,
    description = 'A smartphone.'
},
['phone_earbuds'] = {
    name = 'phone_earbuds',
    label = 'Wireless earbuds',
    weight = 50,
    type = 'item',
    image = 'phone_earbuds.png',
    unique = true,
    useable = true,
    shouldClose = true,
    description = 'Wireless earbuds.'
},
```

Keep `unique = true` on the phone: each handset then carries its own serial in its `info`, which
is what lets a new phone ask for the first-run setup again and keeps Face ID per device. The
phone registers both items as usable through `exports['qs-inventory']:CreateUsableItem`.

## qb-inventory and derivatives

Declare the items in `qb-core/shared/items.lua`, following the shape of the entries already
in that file. The `INSTALL/sql/` files are for the ESX `items` table and do not apply here. The item **names** must match `Config.ItemName` and `Config.Earbuds.itemName`;
the rest (label, weight, image) is yours. `ps-inventory` and `lj-inventory` are detected the
same way.

The phone reads the declared item list from the core, and asks the inventory's exports
(`HasItem`, `GetItemByName`, `SetItemData`) whether a player carries the item: current qb-core
no longer has these functions on the player.

## Custom inventory

Set `Config.Inventory = 'custom'` in `config/bridge.lua` and fill `NashInvCustom` in
`server/bridge/custom.lua`. Two functions are required:

| Function | Returns | Used for |
|---|---|---|
| `hasItem(src, item)` | `boolean` | Can this player open the phone, and keep music in the earbuds |
| `itemExists(name)` | `true`, `false`, or `nil` | The boot check below |

Three more are optional: `serial` and `porteSerial` (recognising a new handset) and
`usableItem` (when your inventory registers usable items itself). See
[Custom Inventory](../compatibility/custom-inventory.md).

`itemExists` returns three values on purpose. `nil` means "cannot conclude" (inventory not
loaded yet, export missing) and the phone stays quiet: a false alarm would send an admin
hunting for a problem that does not exist.

## Giving the item

```
/giveitem <player id> phone 1
```

Works the same on ESX and `ox_inventory`. Otherwise use your shop, your starter kit, or your
inventory's own admin command.

## The boot check

With `Config.UseItem = true`, fifteen seconds after the framework is detected (enough for the
inventory to finish loading its item list), the server verifies that `Config.ItemName` actually
exists. If it does not, it prints this block in red:

```
========================================================================
[nash_phone] Config.UseItem = true but the item "phone" DOES NOT EXIST
[nash_phone] in your inventory -> NOBODY can open the phone.
[nash_phone] The /phone command and the F1 key both check this item on the
[nash_phone] server, so there is no way around it. Pick ONE fix:
[nash_phone]   1. create the item in your inventory (documentation: Installation > Inventory Items)
[nash_phone]      or import INSTALL/sql/item_phone.sql if your framework keeps items in the database.
[nash_phone]   2. or set Config.UseItem = false in config/main.lua to open
[nash_phone]      the phone without any item.
========================================================================
```

When the item exists, nothing is printed. The earbuds item gets the same check, with a yellow
warning instead: without it nobody can pair earbuds, and nothing else changes.

## Opening without an item

```lua
Config.UseItem = false
```

The phone then opens for everyone with `/phone` (`Config.OpenCommand`) or the key
(`Config.OpenKey`, default `F1`), with nothing in their pockets. Useful for a test server, or
for a server that hands phones to every citizen by design.

## What the earbuds do

| Key | Default | Meaning |
|---|---|---|
| `Config.Earbuds.enabled` | `true` | `false` removes the feature entirely and hides the setting |
| `Config.Earbuds.itemName` | `'phone_earbuds'` | Item name, to create in your inventory. Using it is the only way to pair the earbuds, whatever `requireItem` says |
| `Config.Earbuds.requireItem` | `true` | Must the player still carry the item for the earbuds to work after pairing |

Music from the Music app plays out of the speaker by default: everyone nearby hears it. With
earbuds connected, that player alone hears it.

1. The player owns the item.
2. Using it once **pairs** the earbuds with the phone.
3. Using it again connects or disconnects them, like a real pair.
4. The state stays reachable in Settings → Audio output.

The item is never consumed. With `requireItem = true` (recommended), losing or selling the
item sends music back to the speaker, so the object keeps a value. With `false`, once paired
they work forever.

{% hint style="info" %}
Without `xsound` there is no music to route in the first place. The setting stays visible and
simply has nothing to do.
{% endhint %}
