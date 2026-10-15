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
| `auto`, `esx`, `qbcore`, `qbox`, `custom` | `auto`, `ox`, `qs`, `qb`, `framework`, `custom` |

`qs` is qs-inventory; `qb` covers qb-inventory and its ps-inventory and lj-inventory forks.

In `auto`, frameworks are probed in the order **QBOX → QBCore → ESX**. QBOX is tried first on
purpose: many QBOX servers run a `qb-core` compatibility resource alongside, and probing
QBCore first would file them under the wrong bridge. Inventories are probed in the order
**ox_inventory → qs-inventory → qb-inventory → the framework's own**: `ox_inventory` comes
first because when it runs it owns the item list whatever framework is underneath.

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

No. The resource creates its 31 tables at boot and adds new columns and indexes on later
updates without any action from you. `sql/nash_phone.sql` ships the same schema for
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
your inventory, nobody opens the phone: not even an admin. The `/phone` command, the key and
the `ToggleOpen` / `OpenApp` client exports all go through the same server-side check, so there
is no way around it. The server prints a red block in the console 15 seconds after startup when
it detects this case.
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

Two more mappings exist and belong to the camera, not the phone: `RETURN` (Enter) takes a
picture and left `Alt` toggles the aiming cursor. Players can rebind them the same way.

</details>

<details>

<summary>Can players walk and drive while the phone is open?</summary>

Yes. The phone keeps the game's input while it is open, and only turns off combat, jump,
cover, the weapon wheel, entering and leaving vehicles, and free camera look: movement is left
alone.

When a text field inside the phone takes focus, the game stops receiving the keyboard so that
typing does not walk the character, and gets it back when the field is released.

</details>

<details>

<summary>How do I take a player's phone away (jail, cuffs, no-signal zone)?</summary>

One client export:

```lua
exports['nash_phone']:ToggleDisabled(true)   -- confiscated
exports['nash_phone']:ToggleDisabled(false)  -- given back
```

It blocks **every** opening path at once: the command, the key, the inventory item and the
`ToggleOpen` and `OpenApp` exports. `exports['nash_phone']:IsDisabled()` reads the current
state.

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

{% hint style="info" %}
Custom apps are listed in the App Store and the App Library like the stock App Store apps. With
`defaultApp = true` (the default) each character gets the app on their home screen once; if
they remove it or delete it, it does not come back on its own. With `defaultApp = false` the
player gets it from the App Store.
{% endhint %}

</details>

<details>

<summary>Will an application written for LB Phone work?</summary>

Often, yes. The field names of `AddCustomApp` and the export names are deliberately the ones
LB Phone uses, so the app's declaration table and its UI code usually need no change at all.

The obstacle is the resource name the app calls in its own Lua. If you can edit the file,
replace `exports['lb-phone']` with `exports['nash_phone']`: one line. If it is escrowed and
unmodifiable, the separate `nash_lbcompat` gateway answers to the expected name and forwards
the calls.

It is not a magic fix: an app is only as portable as what it actually uses. It will only partly
work, or not at all, if it:

- relies on the functions LB Phone injects into its page (`fetchNui`, `resourceName`, `getSettings`, `onSettingsChange`, or components such as pop-ups, context menus, photo and contact pickers, the camera) without a fallback of its own;
- reads LB Phone's own database tables (for example `phone_phones` or `phone_messages`): NASH Phone stores its data in its own tables, so those queries find nothing;
- calls LB Phone functions NASH Phone has no equivalent for: they return a neutral value, so the matching feature stays empty;
- has a layout made for LB Phone's screen: NASH Phone's status bar and Dynamic Island cover the top 62 px.

Test every third-party app on a test server before promising it to your players. Details:
[Custom Apps, Apps written for LB Phone](developer-api/custom-apps.md#apps-written-for-lb-phone).

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

Each file carries two blocks: `dict` (everything drawn on the phone screen, around 2 750
strings) and `lua` (the messages shown outside the phone: notifications, warnings, key
labels, around 80 strings). Placeholders differ between the two: `{city}` and `{name}` in
`dict`, `%s` and `%.1f` in `lua`: and must be kept intact. See [Add a language](guides/add-language.md).

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

The call rings, displays, connects and is written to the call log: it is simply silent. A
missing voice resource is not an error, the phone just skips the voice part.

pma-voice is the only voice resource the phone drives: it puts each call on its own pma-voice
call channel. Keep `Config.Voice.provider = 'pma-voice'` (the shipped value) in
`config/main.lua`. Any other value leaves calls silent, even if that resource is running.

</details>

<details>

<summary>How many people can be in one call?</summary>

Four, invited participants included. The cap is fixed and physical, not decorative: video
travels as a **mesh**, each participant holding a connection to each other one. At four, each player already maintains three connections and sends three
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
cutting anyone off. Only followers of the host can watch. `Config.Social.Live.audience`
(`'followers'` by default, also `'everyone'` or `'none'`) decides who is **notified** when a
live starts; it applies to every live on the server, and the host does not choose it.

The host can also bring up to three viewers on screen as guests.

</details>

<details>

<summary>Do InstaPic live streams have sound?</summary>

Yes, by default: viewers hear the microphone of the host and of the guests on screen. It is
`Config.Social.Live.microphone` in `config/social.lua`; `false` gives silent lives with no mic
button. The game's own sound is never sent, because the embedded browser never receives GTA's
audio mix. The sound travels with the picture, not through `pma-voice`, so it also needs
`Config.VideoCall.enabled = true`, and a TURN relay for players behind a 4G box, CGNAT, strict
NAT or a VPN.

When a player's microphone is refused, missing or has not answered after 5 seconds, the live
goes on with picture only and a red crossed-out mic button lets them try again.

</details>

<details>

<summary>Do I need a TURN server for video calls and live streams?</summary>

On a public server, yes. `Config.VideoCall.iceServers` ships with Google's public STUN server
only, which works when both players' networks accept a direct connection.

A **TURN** server relays the image when a direct connection is impossible: players behind a
4G box, CGNAT, strict NAT or a VPN. Without one, those players get the audio but no picture;
the rest of the call works normally.

The simplest relay is Cloudflare: create a free account, create a TURN key, and paste its Key
ID and API token into [`config/turn.lua`](config/config-turn.md). The free tier covers the
first 1,000 GB each month. You can also use your own TURN server (coturn, or another provider
with fixed credentials) in `Config.VideoCall.iceServers`. Details in
[Video calls and live streams](guides/setup-video-calls.md).

</details>

<details>

<summary>Does the video stream go through my server?</summary>

No. The server only relays the handful of signalling messages that let the two embedded
browsers find each other: an offer, an answer, connection candidates: a few kilobytes per
call. The image travels directly from one player to the other, or through the TURN relay
(Cloudflare or your own) when no direct route exists.

The server's signalling relay only forwards between participants of a video call that is actually in progress,
or of an InstaPic live in progress. Payloads above 16 KB are refused, and each player is held
to a signalling budget of 512 KB per 10 seconds.

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

Players can still bring a picture from Discord into their phone: see the next question.

</details>

<details>

<summary>Can players import a photo from Discord?</summary>

Yes. In Photos, the **+** button of the **Albums** tab opens an **Add Photo by URL** alert. In
Discord, the player right-clicks the image, picks **Copy Link**, pastes it into the alert and
taps **Add**. The photo appears in the Library and in the **Recently Saved** collection.

The phone does not keep the Discord link: it expires after 24 hours, and `nashphone_cleanup`
would delete it. The player's game downloads the picture while the link still works,
re-encodes it like a camera photo and uploads it to your Fivemanage storage. The saved photo
points at Fivemanage, so it keeps working after the Discord link dies. Your server never
downloads the link itself.

What to tell players:

- **Any direct link to a picture works**, not only Discord: Imgur, Fivemanage, most image
  hosts. JPEG, PNG, GIF and WebP are accepted, up to 15 MB and 8192 pixels per side by default.
- **A Discord link older than 24 hours** is refused with "This link has expired. Copy it again
  from Discord." Copying it again from the message gives a fresh one.
- **A link to a page is not a link to a picture.** "Copy Message Link" in Discord, or an Imgur
  page instead of the `i.imgur.com` address, is refused as not being a photo.
- **An animated GIF keeps only its first frame**, and the photo's metadata (GPS position
  included) is removed.

No **+** button? The server has no image host in `config/upload.lua`, or
`Config.Gallery.Import.Enabled` is `false`. Every setting is in
[config/main.lua](config/config-main.md#adding-photos-by-link).

</details>

<details>

<summary>How do I share a phone photo on Discord?</summary>

Copy its link, then paste it in Discord. Two ways, same result:

- open the photo, tap the **…** button at the top right, then **Copy**;
- or tap **Share**, then **Copy Photo** (**Copy Video** for a video).

A **Link Copied** bubble confirms it. The link goes to the player's clipboard, the same one as
the rest of their computer, so it pastes straight into Discord, where the link of a photo
displays as the picture itself.

Two things worth knowing before players rely on it:

- **The link is public.** Anyone who has it can open the file, with no sign-in. Deleting the
  photo from the phone does not delete the file from Fivemanage.
- **It lasts as long as your Fivemanage account keeps the file.** With no retention policy, the
  file stays until you delete it. With one (7 to 365 days), the file is deleted after that delay
  and the link stops working. See [How long links last](installation/image-hosting.md#how-long-links-last).

</details>

<details>

<summary>Is there a limit on how many photos a player can upload?</summary>

Yes: 24 upload requests per minute per player, which is about **12 photos a minute**, because
each photo, snap or video sends two files (the media and its thumbnail). On top of that, a
player can save at most 20 photos, videos and imports per minute in Photos. Both limits are
fixed: they are not config settings. It exists because removing the API key from the client prevents
reading it, not using it: without a counter, a modified client could drain your Fivemanage
quota without even sending a file.

A photo added by link counts on the same counter: up to two requests per import, one for the
picture and one for its thumbnail.

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

Treat both state bag keys as read-only. Only the phone's server writes them, and it puts back
any value written from anywhere else, a client included.

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

<summary>How do I check that the install is healthy?</summary>

Restart the resource and read the server console from the top. Most lines are in French:

| Line | What it tells you |
|---|---|
| `framework détecté : …` and `inventaire détecté : …` | The bridges retained. When you named them in `config/bridge.lua`, the lines read `framework : … (imposé par la configuration).` and `inventaire : … (imposé par la configuration).` |
| `schéma DB prêt (31 tables…)` | The database answered and the tables are in place |
| `Langue « … » chargee : …` | One green line per language loaded |
| Red block `Config.Upload is NOT configured` | `config/upload.lua` has no key: the camera is off |
| Red block `Config.UseItem = true but the item "…" DOES NOT EXIST` | Nobody can open the phone. Printed 15 seconds after startup |
| Yellow block `config/apps.lua : attention aux consequences ci-dessous.` | An app the others depend on is turned off |

Most red and yellow lines have their own entry in [Common Errors](common-errors.md); the
`config/apps.lua` block is covered under *Can I disable an application?* above. Then test the
phone in game.

Three commands exist for specific jobs:

| Command | Where | What it does |
|---|---|---|
| `/phonepos` | In game | Prints in your F8 console the block to paste into `config/maps.lua` for your current position |
| `/nashprop` | F8 console | Diagnoses the 3D model in hand |
| `nashphone_cleanup` | Server console only | Counts the dead Discord-hosted gallery rows; `nashphone_cleanup confirm` deletes them |

`/phonepos` and `/nashprop` are open to every player: they only print in that player's own
console and change nothing.

</details>

<details>

<summary>Can I log what players do with the phone on Discord?</summary>

Yes, in `config/logs.lua`, a server-only file. Logs go to Discord webhooks, to Fivemanage, or
both. Two kinds exist:

- **Staff logs** carry the player's identity (character name, server ID, number, identifier,
  Discord mention). Nineteen events: posts, stories, live streams, service requests, photos,
  videos, photos added by link, bank transfers, social account creation, Cindr reports, and
  private communications (SMS, private messages, mail, calls), which ship turned off.
- **Public logs** carry the content only, never the identity, to feed roleplay channels. They
  exist for posts, stories, live streams and service requests, and ship turned off.

Logs ship set to Discord (`admin.service = 'discord'`) with every webhook empty, so nothing is
sent until you fill one in. To check a webhook, trigger in game one of the events it receives:
a refused send prints a line in the server console (see
[Common Errors](common-errors.md#logs)). See [config/logs.lua](config/config-logs.md).

</details>

<details>

<summary>Is the phone branded NASH inside the game?</summary>

Only where you leave it. `Config.Brand` in `config/main.lua` replaces the visible brand:
`name` is used for the network operator in Settings → SIM, the account header, the Wi-Fi
network name, the device name, the settings footer, and the sender of a verification code for
an app that has no sender name of its own (the six social apps sign their codes with their own
name). `studio` is the
publisher shown on the App Store page of every app, as written (`'Nash Store'` by default),
and `site` is the bookmark on the browser's start page. `Config.City` in `config/settings.lua`
sets the city name shown in the weather, clock, compass and social apps.

A field left empty or malformed falls back to the shipped value: the phone never shows a hole
on screen.

</details>
