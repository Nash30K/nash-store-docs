# Custom Framework

If you run a private or in-house framework, wire it into NASH Phone by filling in one file:
`server/bridge/custom.lua`. It is the only file in the resource meant to be edited: everything
else can be updated without touching it.

Nothing in this file is used until `Config.Framework` says so.

## Enabling it

```lua
-- config/bridge.lua
Config.Framework = 'custom'
```

`NashFw.custom.detect()` always returns `false`, on purpose: a custom bridge is never picked by
auto-detection, you have to name it. On success the console prints:

```
[nash_phone] framework : custom (imposé par la configuration).
```

The inventory is a **separate** switch. Keeping your framework custom while leaving
`Config.Inventory = 'auto'` (so `ox_inventory` is detected normally) is a perfectly ordinary
setup. See [Custom Inventory](custom-inventory.md).

## The shape of a provider

`server/bridge/custom.lua` registers a table in the global `NashFw`:

```lua
NashFw = NashFw or {}

NashFw.custom = {
    playerLoadedEvent = 'my_core:playerLoaded',   -- or nil
    detect = function() return false end,
    build  = function()
        return {
            identifier = function(src) ... end,
            -- the rest of the interface
        }
    end,
}
```

| Field | Role |
| ----- | ---- |
| `playerLoadedEvent` | Name of the event your framework fires when a character is loaded. May be `nil` |
| `detect` | Only consulted in `'auto'` mode. Leave it returning `false` |
| `build` | Called at the **first use** of the bridge, and returns the table of functions below |

`build()` is where you fetch your core object. Do not fetch it at file load: the resource may
start before your core does. Nothing in `server/bridge/` calls anything while loading, and
that is what makes the file order inside the glob irrelevant.

### `playerLoadedEvent`

The phone listens to the `playerLoadedEvent` of **every registered provider**, without
resolving any of them, and hands the subscriber a bare `source`. Naming your event here is
enough: there is no argument shape to match.

Leave it `nil` if your framework has no such event. The phone then catches up at the first NUI
bootstrap, slightly later: the player is hydrated when they open the phone rather than when
they spawn.

## Only one function is mandatory

`identifier` is the key of **every** phone table. Without it, the bootstrap callback returns
`false` and the phone never opens.

Everything else is optional. A function you leave empty returns a neutral value and prints one
warning, once:

```
[nash_phone] le pont « custom » n'implémente pas getBank() : valeur neutre rendue.
```

The phone still boots, which is what lets you wire the bridge in stages.

| Left empty | What you lose |
| ---------- | ------------- |
| `fullName` | The RP name shows as the translated "Unknown" everywhere |
| `getBank` | The Bank app shows 0 and transfers fail |
| `job` | The Services app recognises no employee |
| `usableItem` | The phone no longer opens from the inventory item, only from the key and `/phone` |
| `group` | No admin command is granted (the `nash_phone.debug` ACE still works) |

## The interface

All of these run **on the server**. `src` is always a server ID.

<details>

<summary>identifier(src): required</summary>

Returns the **stable** identifier of the character, as a string of at most 64 characters (the
`owner` column is a `VARCHAR(64)`). Return `nil` when the player is not resolved yet.

```lua
identifier = function(src)
    return exports['my_core']:GetCharacterId(src)
end,
```

{% hint style="danger" %}
This value must never change for a given character. It is what ties a player to their number,
contacts, messages, photos and social accounts. A session-scoped value: a temporary id, a
source, anything regenerated on connect: gives the player an empty phone every time and leaves
the previous rows orphaned.
{% endhint %}

Called from roughly ninety places in the resource. It is the one function with no survivable
fallback.

</details>

<details>

<summary>getPlayer(src)</summary>

Returns your framework's player object, or `nil`. The phone does not read it: it only passes it
back to other functions of the same bridge, and uses "did it return something" as a
"this player is connected and loaded" test: the Bank app rejects a transfer whose recipient
returns `nil` here.

Neutral value: `nil`.

</details>

<details>

<summary>fullName(src)</summary>

Returns the displayed RP name, or `nil`.

{% hint style="warning" %}
Return `nil` when you do not know: **do not** return "Unknown" yourself. That fallback is
displayed to a reader whose phone may be in another language, so each caller translates
`unknown_name` for its own recipient. Returning your own literal defeats that, and an empty
string prints a hole.
{% endhint %}

Neutral value: `nil`. Used by the Settings profile, mail senders, AirDrop, the share sheet and
Services requests.

</details>

<details>

<summary>group(src)</summary>

Returns the admin group as a string (`'admin'`, `'superadmin'`, …), or `nil` when unknown.

Return `nil`, not `'user'`, while the player is loading: confusing "not known yet" with "plain
player" rejects an admin who has just connected.

Compared against `Config.DebugGroups` (default `{ 'admin', 'superadmin' }`) to gate the
`/phone*` test commands. Neutral value: `nil`.

</details>

<details>

<summary>job(src)</summary>

Returns the player's job in **exactly** this shape, or `nil`:

```lua
{
    name = 'police',        -- string, matched against config/services.lua
    grade = 3,              -- number
    gradeName = 'sergeant', -- string
    label = 'Police',       -- string, displayed
    onDuty = true,          -- optional boolean
}
```

Return `nil`: not an empty table: while the player is not loaded. "Not known yet" is not
"unemployed".

`onDuty` is only read when you set `Config.Services.ownDuty = false` to drive duty from your
own clock-in system. With the shipped default (`true`), the phone owns the Duty switch and
`onDuty` is ignored.

Neutral value: `nil`, which makes the Services app treat every player as having no job.

</details>

<details>

<summary>getBank(src) / getCash(src)</summary>

Return the bank and cash balances as numbers. Both feed the Bank app summary.

Neutral value: `0`.

</details>

<details>

<summary>addBank(src, amount) / removeBank(src, amount)</summary>

Credit and debit the bank account. **Return an honest boolean.**

```lua
addBank = function(src, amount)
    return exports['my_core']:AddMoney(src, 'bank', amount) == true
end,
```

{% hint style="danger" %}
A phone transfer debits the sender, then credits the recipient, then refunds the sender if the
credit failed. Returning `true` when nothing moved makes money disappear: and returning `true`
from `removeBank` without checking the balance creates it. `removeBank` must refuse an
overdraft.
{% endhint %}

Neutral value for both: `false`, which makes every transfer fail cleanly rather than silently.

</details>

<details>

<summary>players()</summary>

Returns your framework's list of connected players, in whatever shape it uses.

Nothing outside the bridge calls it today: the phone enumerates online players with the
native `GetPlayers()` instead, precisely because framework list shapes differ. Leaving it
returning `{}` is safe.

</details>

<details>

<summary>usableItem(name, cb)</summary>

Registers `name` as a usable inventory item. Your implementation must call `cb(src)` when a
player uses it; `cb` opens the phone.

```lua
usableItem = function(name, cb)
    exports['my_core']:RegisterUsableItem(name, function(src)
        cb(src)
    end)
end,
```

Called twice at startup: once for `Config.ItemName` (default `'phone'`), once for
`Config.Earbuds.itemName` (default `'phone_earbuds'`) when earbuds are enabled.

Leaving it empty only removes the "use the item to open the phone" path. `/phone` and the open
key still work, and both still check that the player carries the item when
`Config.UseItem = true`.

</details>

<details>

<summary>invHasItem(src, item) / invItems() / invSerial(src, item) / invPorteSerial(src, item, serial)</summary>

The framework's **native** inventory. These are only read when `Config.Inventory` resolves
to `'framework'`.

- `invHasItem(src, item)` returns whether the player carries at least one. `true`, a number
  greater than zero, and the string `'1'` are all accepted.
- `invItems()` returns the table of declared items keyed by name, or `nil` if you cannot
  provide it: `nil` means "cannot tell", and the phone stays quiet instead of warning.
- `invSerial` and `invPorteSerial` are **optional**, and ship returning `nil`. They let the
  phone tell one handset from another, so that a new phone asks for the first-run setup again.
  A framework's native inventory usually cannot do this, and `nil` is a perfectly good answer:
  the setup then simply never replays. The two functions are documented in full, with their
  pitfalls, under `serial` and `porteSerial` in [Custom Inventory](custom-inventory.md).

If your items are held by `ox_inventory` or another inventory resource, ignore all four and
configure the inventory bridge instead. See [Custom Inventory](custom-inventory.md).

</details>

## What you do not implement

These are built on top of your provider by `server/bridge/framework.lua`, and you should not
try to override them:

| Helper | Built from |
| ------ | ---------- |
| `Framework.hasItem(src, item)` | Delegated to the **inventory** bridge, never to the framework |
| `Framework.itemExists(name)` | Delegated to the **inventory** bridge |
| `Framework.itemSerial(src, item)` | Delegated to the **inventory** bridge |
| `Framework.itemPorteSerial(src, item, serial)` | Delegated to the **inventory** bridge |
| `Framework.hasJob(src, name)` | `job(src).name == name`, strict: no case folding, no trimming |
| `Framework.jobPlayers(name)` | A scan of the native `GetPlayers()`, filtered with `hasJob` |
| `Framework.name()` | The resolved bridge name, reported by `/phonedeps` |
| `Framework.onPlayerLoaded(fn)` | Your `playerLoadedEvent` |

The strictness of `hasJob` is deliberate: a job name misspelled in `config/services.lua` should
be visible, not silently half-matched.

## Registering under your own name

Instead of filling `NashFw.custom`, you can add a provider under its own name, in its own file
inside `server/bridge/`:

```lua
NashFw.myframework = {
    playerLoadedEvent = 'my_core:playerLoaded',
    detect = function() return GetResourceState('my_core') == 'started' end,
    build  = function() return { --[[ ... ]] } end,
}
```

```lua
Config.Framework = 'myframework'
```

{% hint style="warning" %}
Auto-detection walks a hard-coded list of three names (`qbox`, `qbcore`, `esx`). A provider you
register under any other name will **never** be picked by `'auto'`, even with a working
`detect()`. You must name it in `Config.Framework`. A name that matches no registered provider
prints the list of the ones that do exist.
{% endhint %}

A file added under `server/bridge/` is loaded by the existing glob in `fxmanifest.lua`, so no
manifest edit is needed: but it is a new file, and a resource update will not carry it over
unless you keep a copy.

## Checklist

1. `[nash_phone] framework : custom (imposé par la configuration).` appears at start.
2. No `n'implémente pas` warnings remain for functions you meant to fill.
3. `/phonedeps` reports the framework in use and the phone item as declared.
4. The phone opens, and Settings shows a phone number and the character's real RP name.
5. Reconnect: the same character gets **the same number** and the same messages back. This is
   the `identifier` stability test, and it is the one that matters most.
6. Two characters of the same player get **different** numbers, if that is your intent.
7. The Bank app shows the right balance; a transfer between two connected players moves money
   on both sides; a transfer to a player who disconnects mid-way refunds the sender.
8. A player holding a job listed in `config/services.lua` sees their service page.
9. Using the phone item from the inventory opens the phone.
10. An admin passes `/phonehelp`; a plain player is refused.
