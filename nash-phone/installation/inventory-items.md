# Inventory Items

NASH Phone creates **no item**. Your inventory owns the item list, so the two items below have
to be declared on your side before players can carry them.

| Item | Config key | Default | Required |
|---|---|---|---|
| The phone | `Config.ItemName` | `phone` | Yes, while `Config.UseItem = true` |
| Wireless earbuds | `Config.Earbuds.itemName` | `phone_earbuds` | No, the feature is optional |

Both keys are in `config/main.lua`.

{% hint style="danger" %}
With `Config.UseItem = true` (the default), there is **no way around the item**. The `/phone`
command and the open key both go through the same server-side check
(`nash_phone:requestOpen`), and nobody can carry an item that does not exist, not even an
admin. Either declare the item, or set `Config.UseItem = false`.
{% endhint %}

## No `client.export` to add

Unlike most phone scripts, you do not wire a usable-item export. `nash_phone` registers the
handler itself at boot, through the framework bridge:

| Framework | What the phone calls |
|---|---|
| ESX | `ESX.RegisterUsableItem` |
| QBCore | `QBCore.Functions.CreateUseableItem` |
| QBOX | `exports.ox_inventory:RegisterUsableItem` |

You only have to declare the item. Using it opens the phone.

{% hint style="warning" %}
On **QBOX**, usable items belong to `ox_inventory`, not to the core. On a QBOX server without
`ox_inventory`, using the item does nothing. The open key and `/phone` still work for anyone
who owns the item, and the boot diagnostic reports it.
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

The same snippets are in the comments of `config/main.lua` (phone) and
`sql/item_earbuds.sql` (earbuds), with French labels. The `label` is what players read in
their inventory, so translate it to your server's language.

Then:

```
restart ox_inventory
```

## ESX default inventory

The classic ESX inventory reads its items from the `items` table in the database. Two ready
files ship with the resource:

| File | Item |
|---|---|
| `sql/item_phone.sql` | `phone` |
| `sql/item_earbuds.sql` | `phone_earbuds` |

Or run the inserts directly:

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

## qb-inventory and derivatives

Declare the items in `qb-core/shared/items.lua`, following the shape of the entries already
in that file. The item **names** must match `Config.ItemName` and `Config.Earbuds.itemName`;
the rest (label, weight, image) is yours.

The phone reads the declared item list from the core, not from the inventory resource:
derivatives of `qb-inventory` rename their exports often, while `GetItemByName` on the core
is stable.

## Custom inventory

Set `Config.Inventory = 'custom'` in `config/bridge.lua` and fill `NashInvCustom` in
`server/bridge/custom.lua`. The phone asks an inventory only two things:

| Function | Returns | Used for |
|---|---|---|
| `hasItem(src, item)` | `boolean` | Can this player open the phone / use the earbuds |
| `itemExists(name)` | `true`, `false`, or `nil` | The boot check below |

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

Fifteen seconds after start (enough for the inventory to finish loading its item list), the
server verifies that `Config.ItemName` actually exists. If it does not, it prints this block
in red:

```
========================================================================
[nash_phone] Config.UseItem = true but the item "phone" DOES NOT EXIST
[nash_phone] in your inventory -> NOBODY can open the phone.
[nash_phone] The /phone command and the F1 key both check this item on the
[nash_phone] server, so there is no way around it. Pick ONE fix:
[nash_phone]   1. create the item in your inventory. See the "ITEM PHONE"
[nash_phone]      section of config/main.lua (ox_inventory / qb-core examples)
[nash_phone]      or import sql/item_phone.sql if your framework keeps items in the database.
[nash_phone]   2. or set Config.UseItem = false in config/main.lua to open
[nash_phone]      the phone without any item.
========================================================================
```

`/phonecheck` reports the same condition as a blocking issue.

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
| `Config.Earbuds.itemName` | `'phone_earbuds'` | Item name, to create in your inventory |
| `Config.Earbuds.requireItem` | `true` | Must the player still carry the item for the earbuds to work |

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
