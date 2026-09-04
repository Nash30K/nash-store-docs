# FAQ

Answers to the questions server owners ask before and just after installing NASH Phone. Every
answer below was checked against the shipped code. If your problem is a message in the console
or something that visibly does not work, go to [Common Errors](common-errors.md) instead.

## General

<details>

<summary>Is NASH Phone standalone?</summary>

No. Four things are required:

| Requirement | Why |
|---|---|
| A framework | Player identifier, RP name, money, job. ESX, QBCore, QBOX, or your own |
| `oxmysql` | Every table the phone owns |
| `ox_lib` | Client/server callbacks, and the fallback notification style |
| OneSync | Declared as a hard dependency in `fxmanifest.lua` (`dependencies { '/onesync', 'ox_lib', 'oxmysql' }`) |

No framework is declared as a hard dependency, and that is deliberate: a missing hard
dependency stops a resource from starting at all. The framework is resolved at boot and
announced in the console instead. See [Requirements](installation/requirements.md).

</details>

<details>

<summary>Which frameworks and inventories are supported?</summary>

Framework and inventory are chosen **separately**, in `config/bridge.lua`. Both default to
`'auto'`.

| `Config.Framework` | `Config.Inventory` |
|---|---|
| `auto`, `esx`, `qbcore`, `qbox`, `custom` | `auto`, `ox`, `qb`, `framework`, `custom` |

In `auto`, frameworks are probed in the order **QBOX → QBCore → ESX**. QBOX is tried first on
purpose: many QBOX servers run a `qb-core` compatibility resource alongside, and probing
QBCore first would file them under the wrong bridge. Inventories probe `ox_inventory` first,
because when it runs it owns the item list whatever framework is underneath.

The console tells you what was retained:

```
[nash_phone] framework détecté : qbox.
[nash_phone] inventaire détecté : ox.
```

A custom framework or inventory is written in `server/bridge/custom.lua`: see
[Custom Framework](compatibility/custom-framework.md).

</details>

<details>

<summary>Do I have to import the SQL file?</summary>

No. `server/db/schema.lua` creates the 26 tables at boot and adds new columns and indexes on
later updates without any action from you. `sql/nash_phone.sql` ships the same schema for
owners who prefer to prepare the database by hand, but it is not needed.

The two files you *may* need are `sql/item_phone.sql` and `sql/item_earbuds.sql`, and only if
your framework keeps items in a database `items` table (default ESX inventory). With
`ox_inventory` or any modern inventory, declare the items there instead. See
[Inventory items](installation/inventory-items.md).

</details>

<details>

<summary>Can I change framework after the server has opened?</summary>

Not without a migration you write yourself. Every table is keyed on the identifier the
framework hands out, and that identifier has a different shape from one framework to another
(an ESX licence, a QB `citizenid`). Players would find an empty phone, and their numbers,
contacts, messages and photos would become unreachable. This is a decision to make before
opening.

</details>

## Opening the phone

<details>

<summary>Do players need an item to open the phone?</summary>

By default yes: `Config.UseItem = true` and `Config.ItemName = 'phone'` in `config/main.lua`.
Set `Config.UseItem = false` and the phone opens for everyone with the command and the key,
with nothing in their pockets.

{% hint style="warning" %}
The resource creates **no item**. If `Config.UseItem` is true and the item does not exist in
your inventory, nobody opens the phone: not even an admin. The `/phone` command and the key
both go through the same server-side check (`nash_phone:requestOpen` in `server/main.lua`),
so there is no way around it. The server prints a red block in the console at startup when it
detects this case.
{% endhint %}

</details>

<details>

<summary>How do I change the key that opens the phone?</summary>

Two keys in `config/main.lua`:

```lua
Config.OpenKey = 'F1'         -- default key
Config.OpenCommand = 'phone'  -- /phone
```

The key is registered with `RegisterKeyMapping`, so each player can also rebind it from the
FiveM keybinding settings without you changing anything. Changing `Config.OpenKey` only moves
the **default**; a player who already rebound it keeps their own choice.

Two more mappings exist and belong to the camera, not the phone: `RETURN` takes a picture and
left `Alt` toggles the aiming cursor (`client/apps/camera_control.lua`).

</details>

<details>

<summary>Can players walk and drive while the phone is open?</summary>

Yes. Opening the phone calls `SetNuiFocus(true, true)` together with
`SetNuiFocusKeepInput(true)`, then a control-disable loop turns off combat, jump, cover, the
weapon wheel, entering and leaving vehicles, and free camera look: but leaves movement alone.

When a text field inside the phone takes focus, the `phone:inputFocus` callback drops
`KeepInput` so that typing does not walk the character, and restores it when the field is
released.

</details>

<details>

<summary>How do I take a player's phone away (jail, cuffs, no-signal zone)?</summary>

One client export:

```lua
exports['nash_phone']:ToggleDisabled(true)   -- confiscated
exports['nash_phone']:ToggleDisabled(false)  -- given back
```

It blocks **every** opening path at once: the command, the key, the inventory item and the
`ToggleOpen` export. `exports['nash_phone']:IsDisabled()` reads the current state.

</details>

## Applications

<details>

<summary>Can I disable an application?</summary>

Yes, in `config/apps.lua`. `enabled = false` removes the app completely: no icon, absent from
the App Store, the library, the search and the settings, and it is pulled off the home screen
of players who already had it.

Four apps hold others up, and the server prints a yellow warning at boot if you turn one off:

| Disabled app | What you lose |
|---|---|
| `appstore` | Every app marked `store = true` becomes permanently unreachable |
| `settings` | No wallpaper, ringtone, language or account management |
| `phone` | No calls, and the call log is unreachable |
| `messages` | No SMS. Other scripts can still send them, the player just never sees them |

The phone will not stop you. It is your server; it only says so once in the console.

</details>

<details>

<summary>Can I rename an app, or move it to the App Store?</summary>

Both, in `config/apps.lua`:

```lua
ticktok = { enabled = true, store = true, name = 'Chicago Clips' },
```

`name` replaces the translation, so the same name shows in French and in English: which is
what you want for a proper noun. `store = true` means the app is not on the phone at first and
has to be downloaded from the App Store; `store = false` pre-installs it on the home screen.
Anything you leave out keeps its shipped value, and an app missing from the table works
normally. See [Choose which apps are installed](guides/customize-apps.md).

</details>

<details>

<summary>Can I add my own application?</summary>

Yes, two ways:

- **In `config/custom_apps.lua`** (`Config.CustomApps`), for an app with no interface of its
  own: it fires `onUse` / `onServerUse` and nothing more.
- **From your own resource**, with `exports['nash_phone']:AddCustomApp({ … })` called
  client-side once the player is loaded. This is the way to go when your app has a UI, because
  its HTML has to live in your resource.

`Config.CustomAppsSettings.max` caps the number of custom apps at 40, so a buggy resource
looping on `AddCustomApp` cannot flood the home screen. Full walkthrough in
[Add a third-party application](guides/add-custom-app.md).

</details>

<details>

<summary>Will an application written for LB Phone work?</summary>

Often, yes. The field names of `AddCustomApp` and the export names are deliberately the ones
LB Phone uses, so the app's declaration table and its UI code usually need no change at all.

The obstacle is the resource name the app calls in its own Lua. If you can edit the file,
replace `exports['lb-phone']` with `exports['nash_phone']`: one line. If it is escrowed and
unmodifiable, the separate `nash_lbcompat` gateway answers to the expected name and forwards
the calls.

An app is only as portable as the exports it actually uses. Exports that are not wired return
a neutral value and write a console warning, so the app starts without crashing but without
data.

</details>

## Languages

<details>

<summary>Is the language per player or per server?</summary>

Both, and they do different jobs.

- `Config.Locale` (`config/main.lua`, shipped as `'fr'`) is the **server** language. It sets
  the language of the notifications the server pushes outside the phone, and the starting
  language of a phone nobody has touched.
- Each player then overrides it in **Settings → General → Language**, and the choice is kept.

`Config.ExtraLocales` lists the extra languages offered in that menu. French and English are
always there.

</details>

<details>

<summary>Can I add a language?</summary>

Yes, and without rebuilding anything. `locales/` holds one JSON file per language, read at
runtime. Duplicate `locales/en.json`, translate it, and point `Config.Locale` (or add the code
to `Config.ExtraLocales`) at it.

Each file carries two blocks: `dict` (everything drawn on the phone screen, around 2 100
strings) and `lua` (the messages shown outside the phone: notifications, warnings, key
labels). Placeholders differ between the two: `{city}` and `{name}` in `dict`, `%s` and
`%.1f` in `lua`: and must be kept intact. See [Add a language](guides/add-language.md).

A missing string falls back to English, then French, then shows its raw key. Nothing breaks.

</details>

<details>

<summary>Can I edit the shipped French and English?</summary>

Yes: open `locales/fr.json` or `locales/en.json` and change the line. They are read at boot
like any other language file.

One caveat: those two files are **shipped**, so a phone update replaces them. Keep a note of
your changes to reapply them. A language you added yourself is never touched, because no such
file is shipped.

</details>

## Calls, video and live streams

<details>

<summary>Do calls work without pma-voice?</summary>

The call rings, displays, connects and is written to the call log: it is simply silent. The
voice provider is isolated in `client/modules/voice.lua` and checked with `GetResourceState`,
so a missing voice resource degrades to a no-op rather than an error. Set
`Config.Voice.provider` if you use something other than `pma-voice`; `/phonedeps` reports on
the provider you actually configured, not on a hardcoded name.

</details>

<details>

<summary>How many people can be in one call?</summary>

Four, invited participants included (`MAX_PARTS` in `server/apps/phone.lua`). The cap is
physical, not decorative: video travels as a **mesh**, each participant holding a connection
to each other one. At four, each player already maintains three connections and sends three
copies of their stream. Beyond that you would need a video mixing server.

An unanswered call becomes a missed call after 30 seconds.

</details>

<details>

<summary>How many viewers can an InstaPic live stream take?</summary>

Eight by default: `Config.Social.Live.maxViewers` in `config/social.lua`. Same reason as
above: with no media server, the broadcaster sends a **separate** stream to every viewer, so
their upstream bandwidth and CPU scale with the audience. Past roughly ten, the broadcast
degrades for everyone already connected and the broadcaster loses frames in their own game.
Raising the number does not create capacity, it spreads the same capacity thinner.

The stream stays open when the cap is reached: newcomers get a readable refusal instead of
cutting anyone off. `Config.Social.Live.defaultAudience` (`'followers'` by default, also
`'everyone'` or `'none'`) is only the value pre-selected on the launch screen; the player
changes it at broadcast time.

</details>

<details>

<summary>Do I need a TURN server for video calls and live streams?</summary>

Not for most players. `Config.VideoCall.iceServers` ships with Google's public STUN server,
which is enough to let two players find each other in the majority of cases.

A **TURN** server relays the image when a direct connection is impossible: players behind
CGNAT, some 4G routers, corporate networks. Without one, those players get the audio but no
picture; the rest of the call works normally. A commented block in `config/main.lua` shows
where to paste your credentials; keep the UDP, TCP and TLS entries together, as that is what
gets through the most firewalls. Rent one or self-host `coturn`. Details in
[Video calls and live streams](guides/setup-video-calls.md).

</details>

<details>

<summary>Does the video stream go through my server?</summary>

No. The server only relays the handful of signalling messages that let the two embedded
browsers find each other: an offer, an answer, connection candidates: a few kilobytes per
call. The image travels directly from one player to the other.

The relay only forwards between participants of a call that is actually in progress, and only
if that call is a video call. Payloads above 16 KB are refused.

</details>

## Photos and media

<details>

<summary>Where are photos stored? Do I need my own image host?</summary>

Yes, and it is the one file you must fill in. The phone stores a **link** to each picture,
never the picture itself, so an image host is required. `config/upload.lua` takes a
`provider` and an `apiKey`; the only supported provider is Fivemanage (free account, no card).

Until it is filled in, the camera refuses to take photos, warns the player, and the server
prints a red block at boot. `config/upload.lua` is loaded in `server_scripts` and **must stay
there**: every shared script is downloaded into each connecting player's cache and readable
with any tool, which would make your API key public. The client only ever learns a boolean:
"is a host configured, yes or no". See [Set up image hosting](guides/setup-image-hosting.md).

</details>

<details>

<summary>Can I use a Discord webhook instead?</summary>

No, and the mode was removed rather than left in place with a warning. Since December 2023
Discord signs its CDN URLs and they expire after 24 hours
(`.../photo.png?ex=…&is=…&hm=…`). The photos post fine, display perfectly, and the whole
gallery becomes a grid of broken thumbnails the next day. Stripping the parameters does not
help.

If you are migrating from an older install, `nashphone_cleanup` (server console only) counts
the dead Discord-hosted rows, and `nashphone_cleanup confirm` deletes them. The pictures are
gone either way; only the rows remain.

</details>

<details>

<summary>Is there a limit on how many photos a player can upload?</summary>

Twelve uploads per minute per player. The counter is shared between the two callbacks that
cost something (asking for a presigned URL, and the legacy relay), because both are halves of
the same upload. It exists because removing the API key from the client prevents reading it,
not using it: without a counter, a modified client could drain your Fivemanage quota without
even sending a file.

Photo and video sizes are set in `Config.Camera`: `Photo.LongEdgePx = 1920` at quality `0.8`,
and `Recording` at 30 seconds max, 30 fps, 1 200 000 bit/s and a 720 px long edge (accepted
range 240 to 1080). The video resolution is the single setting that weighs most on in-game
performance, because the film is encoded live while GTA V runs.

</details>

## Social networks and privacy

<details>

<summary>Can a player get a list of every account on the server?</summary>

No, and nothing in the social module offers one. To write to someone you have to know their
username and search for it. Accounts can also be set private by their owner, which removes
them from search entirely.

The byCloud sign-in screen answers a wrong password and an unknown address with the **same**
refusal, on purpose: distinguishing them would turn the login screen into a directory. The
same two counters guard sign-in and account creation: 8 failures on one address, or 24
across all addresses, lock that player out for five minutes.

The stories carousel is the one place where a player sees someone they do not follow, and only
when `Config.Social.StoriesAudience` is set to `'everyone'` for that app (shipped:
`instapic = 'everyone'`, `snapz = 'following'`). Even then it only shows people who published
in the last 24 hours, and only because they published. The post **feed** stays limited to
follows in every app, whatever you set here.

</details>

<details>

<summary>Does the Snapz map store where players are?</summary>

No. A position is never written to the database: it is read from the game at the moment the
map opens, and it disappears with the player's connection. Nobody can find out where somebody
stopped playing.

Four locks stand between a player and someone else's dot, in `config/social.lua`:
`requireMutual = true` (both must follow each other), the target's own sharing choice
(never shared until they answer the question), being connected, and `enabled = true`. Set
`blurMeters` above zero to round positions onto a grid: the friend shows in the right
neighbourhood, never at the right door. The blur is applied **server-side**, so a modified
client cannot get anything more precise.

</details>

<details>

<summary>Someone stole another player's byCloud password and read their mail. Is that a bug?</summary>

No, it is the designed behaviour and there is nothing to fix. A byCloud account carries its
owner's mailbox, so whoever knows the address and the password reads the mail, sends from that
address, and recovers the password keychain. Stealing credentials is meant to have
consequences.

`Config.Cloud.crossLogin = false` closes it: an account then only opens on the character that
created it. The trade-off is direct: a player no longer finds their own account on another
character. Accounts created before the owner column existed have no owner recorded and stay
open to everyone even at `false`, because guessing who created them would be worse.

</details>

## Integrating with your own scripts

<details>

<summary>How do I send an SMS from my own resource?</summary>

```lua
local number = exports['nash_phone']:GetPhoneNumber(source)
exports['nash_phone']:SendMessage('555-0100', number, 'Your order is ready')
```

The **first** argument is the sender's *number*, not a source. That is what lets a business
write under its own name. Sender and receiver are clamped to 16 characters (the column width)
rather than refused, and the body is capped at 16 000 characters.

For a banner rather than a text message:

```lua
exports['nash_phone']:SendNotification(source, {
    app = 'wallet', title = 'Transfer received', content = '+$500',
})
```

`target` accepts a source **or** a phone number. The fallback title is rendered in the
recipient's own phone language, not the server's.

</details>

<details>

<summary>How do I read a player's phone number from another resource?</summary>

Either the export or the state bag:

```lua
local number = exports['nash_phone']:GetPhoneNumber(source)
local number = Player(source).state.phoneNumber
```

Both are set when the framework finishes loading the player, not when the phone is first
opened: a player who has never taken their phone out is still reachable. `phoneName` is
published on the same state bag.

</details>

<details>

<summary>Is there a battery system?</summary>

Not a drain. The phone displays 100 % until another resource drives it:

```lua
exports['nash_phone']:SetBattery(12)
exports['nash_phone']:ToggleCharging(true)
local level = exports['nash_phone']:GetBattery()
```

These are **client** exports and the value lives client-side; the server does not know it and
does not overwrite it on bootstrap. Below 20 % and not charging, the status bar icon turns
red; while charging it stays green even below 20 %. There is no low-power mode in this
version.

</details>

<details>

<summary>What does a player see when a message arrives and the phone is put away?</summary>

`Config.NotifyStyle` in `config/main.lua` decides:

| Value | Result |
|---|---|
| `'phone'` (shipped) | The phone's own iOS-style banner, rising from wherever the player parked their device, at the size they gave it |
| `'oxlib'` | A plain `ox_lib` toast, in your server's style |
| `'both'` | Both, useful for comparing before choosing |

The web page is never unloaded: closing the phone releases focus and hides the chassis, but
the page keeps running over the game: which is how the banner can be drawn without the phone
being open.

</details>

## Operations

<details>

<summary>Does the phone cost anything while it is closed?</summary>

The per-frame control-disable loop only runs while the phone is open. The speaker's proximity
scan only runs while a speaker is on. The Snapz map only polls while a player is looking at
it, and stops when the tab closes. Notification cleanup runs at boot and then hourly rather
than continuously.

Per-player settings are held in memory and written to the database in batches 3 seconds later,
so a burst of toggles is one query rather than one query per switch; an identifier idle for 15
minutes leaves memory. Players on lower-end machines can turn `perfBlur` and `perfAnim` off
in the phone's own settings.

</details>

<details>

<summary>What are the diagnostic commands, and who can run them?</summary>

Set `Config.Debug = true` in `config/main.lua`, restart the resource, then:

| Command | What it answers |
|---|---|
| `/phonecheck` | The short verdict: only what stops the phone from working |
| `/phonedeps` | Each dependency's state and what stops working without it |
| `/phoneschema` | Whether the expected tables, columns and indexes are there |
| `/phoneconfig` | The configuration **as the server actually read it** |
| `/phonediag` | One player's phone: number, mail address, accounts, counts |
| `/phonebase` | Row counts per table, all characters combined |
| `/phonehelp` | The full list of test commands |

`/phoneconfig` is the one that pays off most often: it shows the *applied* configuration,
which immediately reveals a malformed block that was silently ignored.

Being an admin is enough: `Config.DebugGroups` ships as `{ 'admin', 'superadmin' }`. The
server console always has the right (type the name without a slash, `phonecheck`). The FiveM
ACE `nash_phone.debug` still works alongside the groups:

```cfg
add_ace identifier.license:YOUR_LICENSE nash_phone.debug allow
```

{% hint style="danger" %}
Put `Config.Debug` back to `false` before opening. The other test commands write to the
database and push events in the player's name: left open, they hand anyone who finds them a
way to mint bank transfers. `Config.Debug` is the real barrier, not the permission check.
{% endhint %}

Two commands work without debug mode: `/phonepos` prints the block to paste into
`config/maps.lua` for your current position, and `nashphone_cleanup` runs from the server
console only.

</details>

<details>

<summary>Is the phone branded NASH inside the game?</summary>

Only where you leave it. `Config.Brand` in `config/main.lua` replaces the visible brand:
`name` is used for the network operator in Settings → SIM, the account header, the device
name, the settings footer and the sender of social verification texts. `studio` is the
publisher shown on App Store pages (a value of `Nash` produces "Nash Arcade", "Nash Labs"),
and `site` is the bookmark on the browser's start page. `Config.City` in `config/settings.lua`
sets the city name shown in the weather, clock, compass and social apps.

A field left empty or malformed falls back to the shipped value: the phone never shows a hole
on screen.

</details>
