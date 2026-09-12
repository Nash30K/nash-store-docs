# config/setup.lua

The sequence of screens that greets a character the **first time** they open their phone, after
buying it in a shop: hello, language, country, appearance, byCloud ID, your identity, data and
privacy, Face ID, passcode.

File: `config/setup.lua`

## How it behaves

It runs **once, per character**. A row is written to `nash_phone_setup` from the very first
screen and the current step is stored in it: a player who disconnects halfway resumes where they
left off instead of starting over from *hello*.

Characters that already existed before this feature was added are marked as configured on the
first startup that follows, and only then: the catch-up pass only runs when the setup table is
empty while phones already exist. Running it on every boot would mark "already configured" a
player who bought their phone yesterday and has not finished.

To see the flow again while testing: `/phonesetupreset`, or delete the character's row in
`nash_phone_setup`. The command only exists with
[`Config.Debug`](config-main.md#test-commands) turned on. Other resources can offer a "brand new
phone" through the `ResetPhoneSetup` export.

When the flow completes, the server fires `nash-phone:setupCompleted` with the character's
choices, for resources that want to hand out a starter item, write a log, or greet the player.
Nothing inside the phone depends on it.

## Global switches

```lua
Config.Setup = {
    enabled = true,
    skippable = false,
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | `false` opens the phone straight onto the lock screen |
| `skippable` | `boolean` | `false` | Can the flow be left unfinished? `true` puts a "Later" button on the language screen and opens the phone with default values |

{% hint style="warning" %}
`skippable` and `passcode.required` contradict each other. Leaving without finishing means
leaving **without a passcode**, and `passcode.required = true` demands one before the server
will save the setup. While both are `true`, the "Later" button is **not shown at all:** no
button beats a button that does nothing. To actually get it: `skippable = true` **and**
`passcode.required = false`.
{% endhint %}

## Hello screen

The centre word is written out progressively, stays readable for about two seconds, fades, and
the next one is written. A full cycle lasts three to five seconds and the list loops.

```lua
greetings = {
    { text = 'hello',   hint = 'Swipe up to open' },
    { text = 'bonjour', hint = 'Balayez vers le haut pour ouvrir' },
    { text = 'hola',    hint = 'Desliza hacia arriba para abrir' },
    { text = 'ciao',    hint = 'Scorri verso l alto per aprire' },
    { text = 'hallo',   hint = 'Nach oben wischen zum Offnen' },
    { text = 'olá',     hint = 'Deslize para cima para abrir' },
    { text = 'привет',  hint = 'Проведите вверх, чтобы открыть' },
},
greetingHoldMs = 2000,
```

| Key | Type | Default | Description |
|---|---|---|---|
| `greetings` | `table` | 7 entries | `text` is the greeting (max 32 characters), `hint` the bottom line in the same language (max 64). `hint` fades in rather than being written |
| `greetingHoldMs` | `number` | `2000` | How long the word stays readable once written, in milliseconds. Values below `600` are raised to `600` |

An empty or unusable list falls back to a single `hello` entry, so the screen is never blank.

{% hint style="info" %}
The handwriting font (Caveat) covers Latin and Cyrillic. A greeting written in another script
(Chinese, Japanese, Arabic) stays perfectly readable, but in the phone's ordinary font: a
stroke cannot be cursive in an alphabet the font does not know. The reveal animation works for
all of them.
{% endhint %}

{% hint style="warning" %}
Write the word in its **real script**. The drawn strokes are looked up by the exact greeting
text: `privet` in Latin letters would not find the stroke for `привет` and would fall back to
the font.
{% endhint %}

## Languages offered

```lua
languages = {
    { code = 'fr', label = 'Francais' },
    { code = 'en', label = 'English' },
    { code = 'de', label = 'Deutsch' },
    { code = 'nl', label = 'Nederlands' },
    { code = 'it', label = 'Italiano' },
    { code = 'es', label = 'Espanol' },
    { code = 'pt', label = 'Portugues' },
    { code = 'ru', label = 'Russkiy' },
},
```

| Field | Type | Limit | Description |
|---|---|---|---|
| `code` | `string` | 8 characters | The language file's code |
| `label` | `string` | 48 characters | Shown as written, in its own language: the only way a player recognises theirs in a list |

{% hint style="warning" %}
This list is **display only**. Picking a language actually switches the phone **only if that
language is really translated** (`locales/<code>.json`). Otherwise the phone stays in its
original language, on purpose: a readable phone beats a half-translated one. The list is mostly
there to make the screen believable.
{% endhint %}

Entries missing a `code` or a `label` are dropped. If nothing survives, the list falls back to a
single `{ code = 'fr', label = 'Francais' }` entry: a language screen with no rows would be a
setup you cannot get out of. The value the phone sends back is checked against this list before
being stored.

## Countries or regions offered

```lua
countries = {
    'Los Santos',
    'Blaine County',
    'Sandy Shores',
    'Paleto Bay',
    'Grapeseed',
    'Vinewood',
},
```

Fill this with **your** universe. It ships with the map's towns rather than two hundred
countries: a server set in Los Santos has no reason to offer Turkmenistan. The list order is used
as written: put it in the order you want on screen. Each entry is capped at 48 characters, and
an empty list falls back to your city name from
[config/settings.lua](config-settings.md).

## byCloud ID

```lua
cloudAccount = {
    enabled = true,
    skippable = true,
},
```

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | `false` removes the account screen entirely |
| `skippable` | `boolean` | `true` | `false` removes the "Set up later" button and makes an account mandatory to continue |

`skippable` ships at `true` because forcing a player to create an account before they can use
their phone is hostile.

This screen is wired to the phone's **real** account system (`nash_phone_icloud`, the same one
the byCloud page in Settings uses): signing in here restores the password keychain **and opens
the account's mailbox**, exactly like a real phone. Whoever knows the address and the password
gets in and reads that mail. It is a roleplay mechanic; it is closed with
[`Config.Cloud.crossLogin`](config-main.md#bycloud-account) in `config/main.lua`.

Signing in and creating an account are offered **side by side**, from the moment the address is
typed. On a server that just opened, no account exists yet: the very first player must be able
to create theirs without having to fail a sign-in first. Nothing to configure: the screen
carries a toggle button permanently.

The ID field arrives **pre-filled** with the character's mail address (`john.doe@ls-mail.com`),
derived from their RP name, using the domain from `Config.Mail.domain`. Pre-filled does not mean
imposed: the player can erase it and type anything, since a byCloud ID is a *login*. What they
cannot do is change their Mail app address this way: that one is derived from the RP name,
frozen at creation, and namesakes are separated by a suffix. That is what stops someone claiming
`police@ls-mail.com` or receiving another player's mail.

{% hint style="danger" %}
At `cloudAccount.skippable = false`, make sure `Config.Setup.skippable` is `true`, or the screen
has **no way out** in one specific case. The server rate-limits a player's sign-in *and*
account-creation attempts: past eight failures on one address, or twenty-four across all
addresses, it refuses everything for five minutes. That is what stops account creation being used
as a directory of the server's accounts. An honest player never reaches those numbers, but if
they do and both `skippable` settings are `false`, their phone is stuck on that screen until the
five minutes are up: they can neither sign in, nor create, nor skip.
{% endhint %}

## Your identity

There is **nothing to configure here.** A screen sits between the byCloud ID and "Data and
privacy" asking for first name, last name and age. All three are attached to the **byCloud
account** and travel with it, like the password keychain: sign in from any phone and they come
back.

It is only shown **when an account is created**, never on sign-in. An account you sign in to
already carries its identity. Asking again would make the player retype, on every new phone,
something entered once: with the risk of overwriting the right value with a typo. The one
exception, and it happens once per account: signing in to an account created before this feature
existed, which has no identity recorded yet.

What the server accepts, with the same figures the screen shows the player: a non-empty first and
last name, at most 64 bytes each (64 plain letters, slightly fewer with accents: it is the
column size), and a whole-number age between **13 and 120**. An out-of-range value is refused,
and nothing is written half-way.

There is no switch here because two switches above already control the screen:
`cloudAccount.enabled = false` removes account creation, so the identity screen too; and
`cloudAccount.skippable = true` (as shipped) means a player who taps "Set up later" has no
account and does not see it either.

## Face ID

```lua
faceId = {
    enabled = true,
    scans = 2,
},
```

| Key | Type | Default | Accepted range | Description |
|---|---|---|---|---|
| `enabled` | `boolean` | `true` | - | `false` removes the Face ID step |
| `scans` | `number` | `2` | 1 – 3 | Number of sweeps around the circle |

Two is the value real phones use, and the one that makes the step believable without dragging it
out. Values outside 1–3 are clamped.

## Passcode

The passcode is real: it feeds the `passcode` and `passcodeEnabled` settings, so the lock screen
the phone already has.

```lua
passcode = {
    required = true,
    length = 6,
},
```

| Key | Type | Default | Accepted values | Description |
|---|---|---|---|---|
| `required` | `boolean` | `true` | - | `false` offers a "Skip" button on the passcode screen |
| `length` | `number` | `6` | `4` or `6` | Passcode length |

{% hint style="warning" %}
`length` accepts **only `4` or `6`**. Any other value: including `8`: silently falls back to
`6`. The setup screen and the server check the exact length together, so a modified client cannot
save a two-digit code.
{% endhint %}

{% hint style="warning" %}
`required = true` cancels `Config.Setup.skippable` at the top of the file. A player who leaves
without finishing leaves without a passcode, which is exactly what this setting forbids: the
server would refuse to save. The "Later" button on the language screen therefore disappears, even
with `skippable = true`, rather than sitting there doing nothing. Pick whichever of the two
matters on your server.
{% endhint %}

With `required = true` there is no way out of the flow without setting a passcode. That is
intended: it is how a real phone behaves. Set `required = false` to offer a "Skip" button.

## What gets written at the end

Finishing the flow writes the player's choices into their normal phone settings in one go:
language, dark mode (from the appearance screen), Face ID, and the passcode with
`passcodeEnabled`. The country and the byCloud address are stored on the setup row.

A value refused at any step writes nothing and interrupts nothing: the next screen still
appears. A setup blocked by one rejected field would be an unusable phone, whereas an unsaved
language is just a setting to redo.

## A new phone asks again

Each handset carries a serial, stamped into the item's own data the first time the phone reads
it. When a player opens a phone whose serial is not the one recorded for their character, the
flow above runs again from `hello`, and the byCloud account is reopened with its password:
exactly like a real handset.

| Situation | What happens |
| --------- | ------------ |
| Stored in a trunk and taken back | Nothing. Same serial, same device |
| Given a different phone item | The setup runs again |
| Phone already owned before this version | Kept as it is. The first serial read is adopted without erasing anything, and only the *next* device counts as new |
| `Config.UseItem = false` | Never replays. No item means no device to recognise |
| Inventory without per instance data | Never replays. See [Custom Inventory](../compatibility/custom-inventory.md) for which inventories can do this |

{% hint style="info" %}
**Only the setup row is reset.** Messages, contacts, the camera roll, the bank history and the
phone number all belong to the character and to their line, not to the handset: a player who
loses their phone does not lose their texts. The byCloud account is not touched either, since it
never belonged to the device.
{% endhint %}

`/phonesetup` prints the recorded serial next to the one currently in hand. That single line
separates "this is a different handset" from "the setup was never saved", which otherwise look
identical from the player's side.

Other resources can react with the `nash-phone:deviceChanged` server event: see
[Server Events](../developer-api/server-events.md).

## Fallbacks

If `config/setup.lua` is missing, or `Config.Setup` is not a table, the server applies
`enabled = true`, `skippable = false`, `greetingHoldMs = 2000` and prints one line in the console
explaining what it expected. Each sub-block (`cloudAccount`, `faceId`, `passcode`, `languages`,
`countries`, `greetings`) falls back independently.
