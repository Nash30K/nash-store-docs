# Commands

NASH Phone registers three kinds of commands: the one players use every day, two client-side diagnostics anyone can run, and a set of 38 test commands that stay locked until you explicitly open them.

## Player commands

<details>

<summary>/phone</summary>

Opens and closes the phone.

```
/phone
```

**Permissions:** none.

**Command name:** `Config.OpenCommand` in `config/main.lua`, default `phone`.

**Key binding:** `Config.OpenKey`, default `F1`, registered through `RegisterKeyMapping` so each player can rebind it in *Settings → Key Bindings*. Setting `Config.OpenKey = ''` registers no binding and leaves only the command.

**Behavior**

- While a call is in progress the command does nothing: hanging up is done with the on-screen buttons.
- Closing is instant and local. Opening goes through the server (`nash_phone:requestOpen`) so the inventory item can be checked.
- If the phone has been confiscated by another script through the `ToggleDisabled` export, the player gets a notification instead of an open phone: the key does not just look broken.

{% hint style="warning" %}
With `Config.UseItem = true` (the default) the item named in `Config.ItemName` must exist in your inventory, or **nobody** can open the phone: the command and the key both check it server-side. The resource prints a red console warning 15 seconds after start when the item is missing.
{% endhint %}

</details>

<details>

<summary>/phonepos</summary>

Prints the player's current position, formatted as a ready-to-paste `config/maps.lua` entry, into the **client** console (F8).

```
/phonepos
```

**Permissions:** none. It only reveals the caller's own coordinates.

**Output:** two lines with the map percentages, then a block:

```lua
{
    id = 'monlieu',
    name = 'Mon lieu',
    category = 'place',
    address = 'Vespucci Boulevard, Del Perro',
    coords = vec3(-1234.5, 456.7, 22.3),
    open24 = true,
},
```

Street names containing an apostrophe are escaped, so the block pastes without breaking the config file. No `subtitle` is emitted on purpose: without it the phone shows the translated category name.

</details>

<details>

<summary>/nashprop</summary>

Diagnoses the 3D phone prop and its colour shell, in the **client** console (F8). This is the first thing to ask for on a "my prop does not show up" ticket.

```
/nashprop
```

**Permissions:** none.

**What it reports**

- State of the `nash-phoneprop` resource
- Requested model name and its `joaat` hash
- Whether the game actually loads the model, and after how long (5 s ceiling)
- The configured fallback model and whether it loads
- The shell / skin settings

If the model does not load, the command names the two usual causes: `nash-phoneprop` not ensured **before** `nash_phone`, or a `stream/` folder missing its `.ydr` / `.ytd` / `.ytyp`.

</details>

<details>

<summary>nashcam_photo / nashcam_cursor</summary>

Camera app key bindings. They are registered as commands so FiveM can expose them in *Settings → Key Bindings*, but they only do something while the camera is active.

| Command | Default key | Effect |
|---|---|---|
| `nashcam_photo` | `Enter` | Takes the picture. Ignored during a call and while a capture is already running |
| `nashcam_cursor` | `Left Alt` | Toggles the mouse cursor over the camera UI |

Both labels go through the locale system, so they appear in the player's own language in the FiveM key bindings screen.

</details>

## Server console commands

<details>

<summary>nashphone_cleanup</summary>

Finds gallery rows whose media was hosted on Discord and removes them. Those links have expired and are unrecoverable: the pictures no longer exist, only the database rows do.

```
nashphone_cleanup            # counts, deletes nothing
nashphone_cleanup confirm    # deletes
```

**Permissions:** **server console only** (`source == 0`). A player typing it in game is refused. It is deliberately not ACE-gated: physical access to the console is the right barrier here, and an ACE would invite a too-permissive setup.

**Behavior:** without `confirm` it reports how many rows are affected and how many players are concerned, and changes nothing. With `confirm` it deletes them.

</details>

## Test commands

`server/dev/` exists to exercise the phone alone, without a second player: it can fabricate an incoming call, a received SMS, a mail, a story, a bank transfer.

### Enabling them

```lua
-- config/main.lua
Config.Debug = true
```

Then restart the resource. On start you get a red console line reminding you the commands are open, plus `/phonehelp` for the list.

{% hint style="danger" %}
Leave `Config.Debug = false` on a live server. These commands write to the database and fire events in the player's name: open, they hand anyone who finds them a way to mint bank transfers and messages. The commands are **registered even when debug is off**, so `/phonehelp` still answers on a live server; it just answers "disabled", in the player's language.
{% endhint %}

### Who can run them

Once `Config.Debug` is on, three doors open: none of which replaces the boolean above:

| Door | Detail |
|---|---|
| Server console | `source == 0` is always allowed. It has no identity and no group, and it already administers the server |
| Framework group | The player's group must be listed in `Config.DebugGroups`, default `{ 'admin', 'superadmin' }`. Resolved through the framework bridge (ESX, QBCore, QBOX) |
| ACE permission | `nash_phone.debug`, for servers that prefer native FiveM permissions: `add_ace identifier.license:XXX nash_phone.debug allow` |

A player who passes none of them gets an explicit refusal, not silence.

{% hint style="info" %}
`Config.DebugGroups = {}` is a valid choice, not a mistake: it means "no group qualifies" and leaves the ACE as the only in-game door. A single group written without braces (`Config.DebugGroups = 'admin'`) is accepted too.
{% endhint %}

### Calls, SMS and mail

| Command | Effect |
|---|---|
| `/phonefakecall [number] [video]` | An incoming call rings on your phone. Add `video` for a video call. Let it ring to get a missed call |
| `/phonecall <number> [video]` | Rings that number in your name if the player is online, otherwise rings your own phone |
| `/phonehangup` | Hangs up any call in progress, real or simulated |
| `/phonefakesms [number]` | An SMS lands on your phone, optionally from a chosen number |
| `/phonesms <number> <text>` | Sends a real SMS to that number |
| `/phoneseedsms [count] [number]` | Seeds a full conversation |
| `/phonefakemail [subject]` | A mail lands in your inbox |
| `/phonemail` | Prints your own mail address |
| `/phonecomms` | State of your calls, messages and mail |

`/phonefakecall` refuses your own number and refuses to run while you are already busy: it does not fabricate a case the real system would never produce. Numbers are capped at 16 characters, the width of the database column.

### Content and device

| Command | Effect |
|---|---|
| `/phoneseed` | Fills your phone: contacts, SMS, photos, bank history, notes, notifications |
| `/phonewipe confirmer` | Erases all your phone data. Without the keyword it only lists what would go |
| `/phonefakebank <amount> [memo]` | Simulates a transfer. A negative amount is a debit |
| `/phonefakenotif <app> [text]` | Delivers a notification for that app |
| `/phonebattery <0-100> [charge]` | Forces the battery level, optionally charging |
| `/phoneearbuds` | Uses your earbuds as if you had the item: pairs, then connects / disconnects |
| `/phonecount` | Counts your rows per table |
| `/phonesetup` | Where your first-run setup stands |
| `/phonesetupreset` | Replays the first-run setup on your next open |

`/phonewipe confirmer` keeps three things: your phone number, your mail address and your cloud account. Everything else goes: contacts, SMS, calls, photos, bank history, notifications, notes, reminders, calendar, home layout, settings, screen time, mail, social accounts. SMS threads are removed from your side only, so your correspondent keeps their copy.

`/phonebattery` and `/phoneearbuds` act on *your* phone and refuse to run from the console.

`/phonefakenotif` accepts any app id. One that is not declared in `config/apps.lua` still produces a notification, without an app icon: the command says so. If nothing appears, check *Settings → Notifications*: the per-app switch there silences everything.

### Diagnostics

| Command | Effect |
|---|---|
| `/phonecheck` | Quick verdict: what is stopping the phone from working |
| `/phonediag [server id]` | Full report, app by app. From the console, pass the player's server id |
| `/phoneschema` | Verifies expected tables, columns and indexes in the database |
| `/phonedeps` | State of the dependencies, and what stops working without each |
| `/phoneconfig` | Summary of the configuration as the server actually read it |
| `/phonebase` | Row count per table, all characters combined |

`/phonecheck` and `/phonedeps` are the two to run first on any install ticket: the former names the blocker, the latter tells you which optional resource is missing and what it costs you.

### Social networks

`<app>` is a social app id: `snapz`, `instapic`, `birdby`, `ticktok` or `chatapp`, depending on what the app supports (posts, stories and DMs do not cover the same set).

| Command | Effect |
|---|---|
| `/phonefakemsg <app> [text]` | An incoming private message |
| `/phonefakepost <app> [text]` | A post from someone you follow |
| `/phonefakestory <app> [text]` | An incoming story |
| `/phonefakelike <app>` | Someone likes your latest post |
| `/phonefakecomment <app> [text]` | Someone comments on your latest post |
| `/phonefakefollow <app>` | A new follower |
| `/phonesocialseed <app> [count]` | Fills a feed |
| `/phonesocial` | Your social accounts and their usernames |
| `/phonesocialnotifs <app> [days]` | Ages social notifications so you can watch the cleanup run |
| `/phonesocialnotifs purge` | Runs the cleanup now |
| `/phonesocialwipe` | Erases everything the seeded test characters wrote |
| `/phonesnapmap [username]` | Explains why the Snapz map does or does not show someone |
| `/phonelive` | State of the InstaPic live streams in progress |
| `/phonelivewho` | Who would be notified if you went live right now |

{% hint style="warning" %}
`/phonefakepost`, `/phonefakestory` and `/phonefakemsg` fire the real public events (`nash-phone:social:posted`, `storyPosted`, `messageSent`) with a seeded author. If you have a Discord logging handler wired to those events, it will record entries no real player wrote. See [Server Events](server-events.md).
{% endhint %}

`/phonesocialwipe` deletes what the seeded characters wrote, on both of their references: their username and their phone number, since on `chatapp` they write under the number. It does not touch what real players wrote.

## Adding your own command

The phone uses plain `RegisterCommand`. To gate one behind the same admin check the test commands use, call the public export rather than re-implementing the group lookup:

```lua
RegisterCommand('givephone', function(source, args)
    if source ~= 0 and not IsPlayerAceAllowed(source, 'nash_phone.debug') then return end

    local target = tonumber(args[1])
    if not target then return end

    local number = exports.nash_phone:GetEquippedPhoneNumber(target)
    print(('%s has number %s'):format(target, number))
end, false)
```

See [Server Exports](server-exports.md) for the full list.
