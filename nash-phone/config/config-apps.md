# config/apps.lua

Which of the 32 applications exist on your server, what they are called, and where they start.

File: `config/apps.lua`

## The three keys

Each application accepts up to three keys, all optional.

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | `false` makes the app **disappear**: no icon, absent from the App Store, the app library, search and Settings. It is also removed from the home screen of players who already had it |
| `name` | `string` | the translated name | Display name shown instead of the original one |
| `store` | `boolean` | per app, see below | `true` = not on the phone at first, the player downloads it from the App Store. `false` = pre-installed on the home screen from the first boot |

Anything you leave out keeps its original value. An application **absent from the table works
normally**, so you can delete the lines you do not care about: including apps added by a later
update that are not yet in your file.

{% hint style="info" %}
`name` replaces the translation, so the same name shows in French and in English. That is
intended: *Chicago Taxi* has no reason to be translated.
{% endhint %}

## Read before disabling

Some applications make the others usable. Turning them off is not forbidden, but know what you
lose:

| App | What breaks |
|---|---|
| `appstore` | No app marked `store = true` can be installed any more. They become permanently unreachable |
| `settings` | The player can no longer change wallpaper, ringtone or language, nor manage their account |
| `phone` | No calls, and the call log becomes unreachable |
| `messages` | No text messages. Other scripts can still send them; the player will not see them |

The phone will not stop you: it is your server. It prints a warning block in the server console
at startup, listing the consequences, including how many `store` apps disabling `appstore` just
made unreachable.

## Dock

The four applications always within reach at the bottom of the screen.

| Identifier | Name | Shipped |
|---|---|---|
| `phone` | Phone | pre-installed |
| `messages` | Messages | pre-installed |
| `camera` | Camera | pre-installed |
| `wallet` | Wallet | pre-installed |

## Pre-installed

On the home screen from the first boot.

| Identifier | Name | Identifier | Name |
|---|---|---|---|
| `contacts` | Contacts | `music` | Music |
| `gallery` | Photos | `safari` | Browser |
| `mail` | Mail | `health` | Health |
| `notes` | Notes | `reminders` | Reminders |
| `maps` | Maps | `passwords` | Passwords |
| `weather` | Weather | `services` | Services |
| `clock` | Clock | `appstore` | App Store |
| `calendar` | Calendar | `settings` | Settings |
| `facetime` | VideoCall | | |

## Downloaded from the App Store

Shipped with `store = true`. Set `store = false` to pre-install one on the home screen.

| Identifier | Name | Identifier | Name |
|---|---|---|---|
| `calculator` | Calculator | `ticktok` | TickTok |
| `compass` | Compass | `instapic` | InstaPic |
| `stocks` | Stocks | `playtube` | PlayTube |
| `snake` | Snake | `snapz` | Snapz |
| `tictactoe` | Tic-Tac-Toe | `chatapp` | ChatApp |
| | | `birdby` | BirdBy |

## Examples

```lua
Config.Apps = {
    -- Turn an app off entirely
    stocks = { enabled = false },

    -- Rename one, in every language
    services = { enabled = true, name = 'Chicago Services' },

    -- Pre-install a social network instead of making players download it
    chatapp = { enabled = true, store = false },

    -- Move a stock app into the App Store
    weather = { enabled = true, store = true },
}
```

## How the settings reach the phone

The server sends the interface a **delta**, not a catalogue: only the applications whose
settings differ from the original are transmitted at bootstrap. `enabled` is only sent when it
is explicitly `false`; `store` is sent in both directions, since it serves as much to take an
app out of the App Store as to put one in.

Entries are ignored rather than passed on when they cannot be used: a key that is not a non-empty
string, a value that is not a table, a `name` that is empty after trimming. The result is cached
at startup: `Config` is not re-read while the resource runs, so a change here needs a
`restart nash_phone`.
