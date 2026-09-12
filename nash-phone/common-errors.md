# Common Errors

Fast fixes for the problems we see most. Each entry lists the symptom, the cause, and the fix.
Where the phone prints a message, that exact message is the heading.

{% hint style="info" %}
Before reading further: set `Config.Debug = true` in `config/main.lua`, restart the resource
and run **`/phonecheck`**. It prints only what stops the phone from working, and it catches
most of what is on this page in one line. Put `Config.Debug` back to `false` afterwards.
{% endhint %}

## Silent failures: read these first

Three things in NASH Phone fail **without any error message at all**. They account for most
support tickets, and none of them is visible in a code editor, in a build, or in the browser
preview.

### An NUI action does nothing in game, but works in the browser preview

**Symptom:** a screen is complete on the server side, the browser preview answers, the
TypeScript compiles, and in game the screen stays empty. Nothing appears in F8, nothing
appears in the server console.

**Cause:** the action is missing from the `FORWARD` table in `client/bridge/nui.lua`. That
table is the list of NUI actions the client relays to the server callback of the same name.
An action absent from it **never leaves the client**. The browser preview still answers
because its mock router does not go through this file at all, which is exactly what makes the
trap invisible.

**Fix:** add the `Protocol.*` entry to `FORWARD`:

```lua
local FORWARD = {
    Protocol.bootstrap,
    -- …
    Protocol.myNewAction,
}
```

Only **request/response** actions belong there. Server pushes (`messagesIncoming`,
`socialIncoming`, `mailIncoming`, `bankIncoming`, `liveState`, `liveFeed`, `servicesUpdate`,
`rtcIncoming`) must stay out: listing one would register a `RegisterNUICallback` that nothing
ever calls. Client-only callbacks (Maps, Compass, the AirDrop peer list) are registered by
their own client file and are not in `FORWARD` either.

Two shipped features were already caught by this during development, the mail app and the
Snapz map. It shows up nowhere except in game, which is why `client/bridge/nui.lua` carries a
warning comment above the Snapz entries.

### A new column is missing on a server that was already running

**Symptom:** one app answers with an error on a live server, while a fresh install of the
same version works. `/phoneschema` reports missing columns.

**Cause:** `CREATE TABLE IF NOT EXISTS` does nothing to a table that already exists. A column
added after first release therefore never reaches an install that is already running, and only
those installs.

**Fix:** the column must be listed in the `COLUMNS` table of `server/db/schema.lua`, which
is replayed at every boot and adds only what `information_schema` says is missing:

```lua
local COLUMNS = {
    { 'nash_phone_calls', 'app_id', "VARCHAR(32) NOT NULL DEFAULT 'phone'" },
    -- …
}
```

If you hit this after updating the resource, restarting is enough: the migration runs at boot
and prints what it added. The same rule applies to the `INDEXES` table just below it.

```
[nash_phone] schéma DB prêt (27 tables, 3 colonne(s) ajoutée(s), 2 index ajouté(s)).
```

### A CSS rule works in the preview and is ignored in game

**Symptom:** a layout, a colour or a gradient is right in the browser and wrong (or missing)
in game. No error anywhere.

**Cause:** the FiveM in-game browser is a **Chromium 103**. It does not know `:has()`
(Chromium 105), `color-mix()` (111), `oklch()` (111), range media-query syntax, the `lh` unit,
or `field-sizing` (123). A declaration it does not understand is **dropped in silence**, and
so is the whole rule when the selector is unknown.

**Fix:** write the resolved value by hand rather than letting the browser compute it: a
hand-written tint instead of `color-mix()`, a class set from the TSX instead of `:has()`, a
pixel `min-height` instead of `2lh`. Test in game, not only in the preview.

Two related rendering rules from the same engine:

- A **composited** layer only receives a **rectangular** clip from its parent. A rounded
  parent containing a `backdrop-filter` or a `transform` layer paints black corners unless the
  child carries its own `clip-path`.
- `transition` is a shorthand: writing it after other transition properties silently cancels
  them.

## Startup

### `[nash_phone] AUCUN framework détecté. Le téléphone ne pourra identifier personne.`

**Cause:** `Config.Framework = 'auto'` and none of `qbx_core`, `qb-core`, `es_extended` was
`started` when `nash_phone` booted.

**Fix**

1. Start your framework **before** `nash_phone` in `server.cfg`:

    ```cfg
    ensure es_extended
    ensure oxmysql
    ensure ox_lib
    ensure nash-phoneprop
    ensure nash_phone
    ```

2. Or name it explicitly in `config/bridge.lua`:

    ```lua
    Config.Framework = 'esx'  -- or 'qbcore' / 'qbox' / 'custom'
    ```

A custom framework goes in `server/bridge/custom.lua`: see
[Custom Framework](compatibility/custom-framework.md).

### `[nash_phone] Config.Framework = "…" : aucun pont de ce nom. Ponts disponibles : …`

**Cause:** a typo in `Config.Framework`. Accepted values are `auto`, `esx`, `qbcore`, `qbox`
and `custom`. `qb`, `qbcore2`, `ESX` and the like are not.

**Fix:** correct the value, or set it back to `'auto'` and read the detected bridge in the
console:

```
[nash_phone] framework détecté : qbox.
```

### `[nash_phone] Config.Inventory = "…" : inconnu. Attendu : auto, ox, qb, framework, custom.`

**Cause:** same typo, on the inventory side. `ox_inventory` is not a valid value; the value
is `ox`.

**Fix:** use one of `auto`, `ox`, `qb`, `framework`, `custom` in `config/bridge.lua`.

### `[nash_phone] Config.Inventory = "custom" mais server/bridge/custom.lua ne définit pas NashInvCustom.`

**Cause:** you asked for a custom inventory bridge but the global it expects was never
declared.

**Fix:** declare `NashInvCustom` in `server/bridge/custom.lua`. The phone only asks an
inventory two things: does this player hold the phone item, and does that item exist on the
server. See [Custom Inventory](compatibility/custom-inventory.md).

### `[nash_phone] le pont « … » n'implémente pas …() : valeur neutre rendue.`

**Cause:** your custom bridge does not implement a function the phone calls. It is a warning,
not a crash: the phone returns a neutral value and carries on.

**Fix:** implement the named function. Each missing one costs a feature: without
`Framework.group` the debug commands only answer the ACE, without `Framework.hasItem` the item
check cannot pass, without money functions the Wallet transfers cannot settle.

### `SCRIPT ERROR: server/main.lua:71: attempt to index a nil value (global 'Framework')`

**Cause:** a stale `fxmanifest.lua` left on the server, or a glob that no longer returns what
it used to, so `server/bridge/**.lua` did not load before `server/main.lua`. The tell-tale
sign is the same console printing `framework détecté : esx` a second later: the table existed,
it simply arrived too late for that line.

**Fix:** replace `fxmanifest.lua` with the shipped one and restart. The current file loads
`server/bridge/**.lua` first and defers the subscription into a `CreateThread`, so the
dependency no longer exists; an old manifest reintroduces it.

## Opening the phone

### `[nash_phone] Config.UseItem = true but the item "phone" DOES NOT EXIST`

The full block is red and six lines long. It appears 15 seconds after boot, once the inventory
has finished loading its item list.

**Cause:** `Config.UseItem = true` but `Config.ItemName` matches nothing in your inventory.
The resource creates no item.

**Fix:** pick one:

1. Declare the item in your inventory. `ox_inventory/data/items.lua`:

    ```lua
    ['phone'] = {
        label = 'Phone',
        weight = 190,
        stack = false,
        close = true,
        client = { image = 'phone.png' },
    },
    ```

    `qb-core/shared/items.lua` for QB. For the default ESX inventory (items in a database
    `items` table), import `sql/item_phone.sql` and **restart `es_extended`:** ESX only reads
    that table at boot.

2. Or open the phone without any item:

    ```lua
    Config.UseItem = false
    ```

{% hint style="danger" %}
There is no way around this check. `/phone` and the key both go through the same server-side
verification (`nash_phone:requestOpen`), and nobody can hold an item that does not exist: not
even an admin.
{% endhint %}

### Pressing the key does nothing, or the player is told "You don't have a phone."

**Cause:** three possibilities, in order of frequency:

1. The player does not hold `Config.ItemName`. Give it: `/giveitem <id> phone 1`.
2. The item does not exist at all: see the entry above.
3. The phone was confiscated by another resource through
   `exports['nash_phone']:ToggleDisabled(true)` (jail, cuffs, no-signal zone). The player then
   gets "Your phone is out of order." instead of silence.

**Fix:** `/phonediag <server id>` reports whether that player has a number and what the
server sees. `exports['nash_phone']:IsDisabled()` reads the confiscation flag client-side.

The key also does nothing on purpose while a call is on screen (the on-screen buttons take
over), and the phone closes itself when the player dies.

### Typing in the phone moves the character

**Cause:** the `phone:inputFocus` callback did not fire, or another resource re-asserted
`SetNuiFocus` without `SetNuiFocusKeepInput`.

**Fix:** the phone opens with `SetNuiFocus(true, true)` **and**
`SetNuiFocusKeepInput(true)` so the player can walk, and drops `KeepInput` only while a text
field holds focus. Every path that restores focus must re-assert `KeepInput(true)`; if you
patched `client/core/open_close.lua` or a screenshot / camera path, that is the line to check.

## Database

### `[nash_phone] index ... sur ... non créé :`

**Cause:** an `ALTER TABLE … ADD INDEX` failed. Usually a permissions issue, or a table too
large for the operation to complete in the boot window.

**Fix:** nothing breaks: a missing index makes lists slower, it does not make them wrong. Run
`/phoneschema` to see which one is missing and add it by hand, for example:

```sql
ALTER TABLE nash_phone_messages ADD INDEX pair_at (`sender`,`receiver`,`created_at`);
```

{% hint style="warning" %}
On an install that has been running for months with millions of rows, the **first** boot after
an update that adds indexes will be noticeably longer. Later boots pay nothing: the migration
probes `information_schema` first and skips what is already there.
{% endhint %}

### Editing a row in `nash_phone_settings` has no effect, or is overwritten

**Symptom:** you change a player's settings in SQL while they are connected, and the phone
ignores it or writes the old values back a moment later.

**Cause:** settings live in **memory** and the memory copy is authoritative; the database is
only a backup, written in batches 3 seconds after the last change. An identifier stays cached
until 15 minutes without access.

**Fix:** do it while the player is offline and their identifier has left the cache, or
restart the resource after the SQL. Same rule for `_mutedThreads`, which lives in the same
row.

This is also how you clear a forgotten passcode: with the player offline, delete their row in
`nash_phone_settings` (the phone falls back to factory settings), then restart the resource.

### Notes, Reminders, social accounts or the keychain come back after being deleted

**Cause:** those apps also keep a copy in the game browser's `localStorage`, which belongs to
the **client**, not to the character. Deleting the database rows behind the running page means
the first edit made in the app writes the local copy straight back.

**Fix:** restart the resource (or have the player reconnect) after clearing the rows. The
local copy is wiped automatically on a **character change**, so it never leaks from one
character to another; it is not wiped by a manual SQL delete.

### An app that was open shows stale data after a test command

**Cause:** most apps read their data once, when they mount. Notes, Calendar and Reminders
only read the server once per phone load.

**Fix:** close and reopen the app. For Notes, Calendar and Reminders, restart the resource if
they were opened earlier in the same session.

## Camera, photos and video

### `[nash_phone] Config.Upload is NOT configured -> the CAMERA IS DISABLED.`

**Cause:** `config/upload.lua` has no `apiKey`, or `provider` is not `'fivemanage'`. The
phone stores a **link** to each picture, never the picture, so no host means no photo.

**Fix:** create a free Fivemanage account, generate a **media** token, paste it into
`config/upload.lua`, restart:

```lua
Config.Upload = {
    provider = 'fivemanage',
    apiKey = 'your-token-here',
}
```

{% hint style="danger" %}
`config/upload.lua` is declared in `server_scripts` and must stay there. Every `shared_script`
is downloaded into each connecting player's cache and readable with any tool: moved up, your
API key becomes public and anyone can spend your quota. The client never receives the key,
only a boolean saying whether a host is configured.
{% endhint %}

### "Capture failed." on every photo

**Cause:** `screenshot-basic` is not started. The camera hides the phone, asks
`screenshot-basic` for the frame, and posts it from that resource's own page.

**Fix:** `ensure screenshot-basic`. `/phonedeps` tells you in one line whether the problem is
the screenshot resource or the upload host: the two failure modes look identical from in
game.

### `[nash_phone] envoi fivemanage refusé (HTTP 401)` / `presign fivemanage: refus (…)`

**Cause:** the API token is wrong, revoked, or is not a **media** token. A `429` means the
account quota is exhausted.

**Fix:** regenerate a media token in the Fivemanage dashboard and paste it into
`config/upload.lua`. The player-facing side of the same failure is a short error code
(`presign_refused`, `presign_bad_response`, `http_401`); the readable detail is only in the
server console.

### The gallery is a grid of broken thumbnails

**Cause:** two possibilities.

1. The pictures were hosted on a Discord webhook. Since December 2023 Discord signs its CDN
   URLs and they expire after 24 hours. The links are dead and unrecoverable: the expired
   address does not carry the message id that would let anyone request a fresh one.
2. The pictures are on Fivemanage, but a **retention policy** on your Fivemanage account
   deleted the older files (7 to 365 days, depending on what was set). The recent photos still
   display; the old ones show a crossed-out picture, and "Image Unavailable" once opened.

**Fix:** nothing brings the photos back in either case. For the Discord rows, from the
**server console** only:

```
nashphone_cleanup            counts and shows a sample, deletes nothing
nashphone_cleanup confirm    deletes the dead rows
```

New photos go to Fivemanage, which returns an address with no signature and no expiry of its
own. To stop the second case from happening again, turn retention off in your Fivemanage
dashboard. See [How long links last](installation/image-hosting.md#how-long-links-last).

### A player says photos stop working after a burst

**Cause:** the upload limiter: 12 uploads per minute per player, shared between the presigned
and relay paths. It exists because removing the API key from the client stops it being read,
not used: without a counter, a modified client could drain your quota without sending a
single file. Photos added by link count too, up to two uploads each (the picture and its
thumbnail).

**Fix:** expected behaviour; wait a minute. If you need to change it, the constants are
`UPLOAD_MAX`, `UPLOAD_WINDOW` and the two `COST_*` values in `server/services/upload.lua`.

### The + button is missing in Photos

The **+** button that adds a photo from a link sits at the top of the **Albums** tab
(Collections in French), not on the Library tab.

**Cause:** the phone hides it on purpose in two cases:

1. no image host is configured in `config/upload.lua`: a pasted link would have nowhere to be
   re-hosted, and the phone never stores the link itself;
2. `Config.Gallery.Import.Enabled = false` in `config/main.lua`.

**Fix:** fill in `config/upload.lua` (see [Image Hosting](installation/image-hosting.md)), or
set `Enabled` back to `true`, then restart the resource.

### Adding a photo by link fails

When a link cannot be added, the phone shows a **Couldn't Add Photo** alert with one of the
messages below. The picture is downloaded and checked by the player's own game, not by your
server, so most of these failures never reach the server at all.

| Message | Cause | Fix |
|---|---|---|
| *(the **Add** button stays grey)* | What was pasted is not a web link. The alert does not let it through, so no message appears | Paste the full link of the picture |
| Adding photos by link is turned off on this server. | `Config.Gallery.Import.Enabled = false` (the **+** button is normally hidden in that case) | Expected if you turned it off. Otherwise set it to `true` and restart |
| No photo host is set up on this server. | `config/upload.lua` has no API key, so there is nowhere to re-host the picture | Fill in `config/upload.lua` |
| The image couldn't be downloaded. Check the link. | The player's game could not download the address: a mistyped or cut link, a host that is down, a page that needs a sign-in, a download longer than 20 seconds, or a host blocked on that player's network or in their country | Open the link in a browser outside the game. If it does not show the picture on its own, it cannot be added |
| This link has expired. Copy it again from Discord. | A Discord link older than 24 hours: Discord answers 403, 404 or 410. It only refreshes its links inside its own app | In Discord, right-click the image, **Copy Link**, and paste the fresh link |
| This image was removed by its host. | Imgur redirected to its "removed" placeholder. Only Imgur's placeholder is recognised: another host that answers 404 or 410 gives "The image couldn't be downloaded" | The picture no longer exists at that address |
| This link doesn't lead to a photo (JPEG, PNG, GIF or WebP). | The address answers something other than a picture: a web page (Discord's **Copy Message Link**, an Imgur page rather than `i.imgur.com`), a video, an SVG. The format is read from the content of the file, not from its name | Use the direct link to the picture: in Discord, right-click the image and pick **Copy Link** |
| This image is too large (15 MB max). | The file is heavier than `Config.Gallery.Import.MaxMegabytes`. The number in the message follows your setting | Use a lighter version of the picture, or raise the limit |
| This image is too big (max 8,192 pixels per side and 50 million pixels in total). | The width or height is above `Config.Gallery.Import.MaxSidePx` (the number follows your setting), or the picture has more than 50 million pixels in total, whatever that setting says | Use a smaller version. Raising the limit gains nothing visible: every picture is brought down to `Config.Camera.Photo.LongEdgePx` anyway |
| Uploading to the host failed. Try again. | The upload to Fivemanage failed: a network hiccup, a refused or revoked token, an exhausted account | Try again. If it fails every time, check the token as for the Camera (see the `presign fivemanage` entry above) |
| Too many photos at once. Try again in a minute. | The rate limit, shared with the Camera | Wait a minute |
| Something went wrong. Try again. | Anything not covered above, including the server refusing to save the final address | Try again. If every import fails this way while the Camera works, see the next entry |

### Every import ends on "Something went wrong. Try again.", but the Camera works

**Cause:** your Fivemanage account serves files from a **custom domain**, and that domain is
not listed in `Config.Gallery.Import.AllowedHosts`. The picture is uploaded, then the server
refuses to save an address on a host it does not know. The Camera does not go through that
check, which is why it keeps working. An entry the server could not read has the same effect:
it lists ignored entries in the console at startup
(`Config.Gallery.Import.AllowedHosts : entree(s) ignoree(s) : …`).

**Fix:** add the domain in `config/main.lua`, keep `fivemanage.com`, and restart:

```lua
Config.Gallery = {
    Import = {
        -- ...
        AllowedHosts = { 'fivemanage.com', 'media.yourserver.com' },
    },
}
```

The files uploaded by the failed attempts stay in your Fivemanage storage, unused; delete them
from the dashboard if you care about the space. See
[AllowedHosts and a custom Fivemanage domain](config/config-main.md#allowedhosts-and-a-custom-fivemanage-domain).

### "Couldn't Copy" when copying a photo link

**Cause:** the link did not reach the clipboard. The phone only shows **Link Copied** when the
copy really happened, so this bubble means there is nothing to paste.

**Fix:** try again. Nothing is lost: the photo and its link stay in the camera roll.

## The 3D model

### `[nash_phone] modèle « custom_phone_prop » introuvable, repli sur « prop_npc_phone_02 ». La ressource du prop est-elle démarrée AVANT nash_phone ?`

**Cause:** `nash-phoneprop` started **after** `nash_phone`, or is not started at all. The
model was not registered when the phone booted, so it fell back to a GTA prop.

**Fix**

```cfg
ensure nash-phoneprop
ensure nash_phone
```

`/nashprop` (client console) prints the full diagnosis: the prop resource state, the requested
model and its hash, whether `HasModelLoaded` succeeded and after how long, the configured
fallback, and the state of each colour shell. Note that `IsModelValid` often answers "no" for
a `DLC_ITYP_REQUEST` model and is only informative there.

### The phone in hand is two-tone (coloured body, orange camera block)

**Cause:** `assets/prop/` did not arrive in full. The body and the light face are flat
colours built in memory from the hex values in `Config.Prop.skin.colors`; the camera block is
a **real image** (`back_<colour>.png`) that has to be present.

**Fix:** copy `assets/prop/` again in full. The console names the missing file:

```
[nash_phone] coque silver : image introuvable (assets/prop/back_silver.png), dos laissé d'origine
```

`assets/prop/*.png` must also stay declared in the `files` block of `fxmanifest.lua`, or the
file is never served to the client.

### `[nash_phone] coque … : couleur invalide (body=…, light=…), coloris ignoré`

**Cause:** a `body` or `light` value in `Config.Prop.skin.colors` is not a valid hex colour.

**Fix:** use `#rrggbb`. And keep the pair's rule: **`light` must always be lighter than
`body`.** It is the flat colour of the faces catching the daylight; if it drops below its
`body`, the area meant to lighten darkens instead and the chassis reads inside-out. Nothing
breaks and nothing warns: the prop just looks dull.

### The colour a player picks is not the colour other players see

**Not a bug.** The texture swap is **local**: on that player's screen every phone takes their
colour, including other people's. Changing a model for all players would cost far more than
the feature is worth. This is documented and deliberate.

## Custom applications

### `[nash_phone] app personnalisée refusée (…) : ui = '…' n'a pas de nom de ressource ; attendu '…/…'`

**Cause:** the `ui` path does not start with your resource name. The iframe is loaded as
`nui://<ui>`, so `ui/index.html` would point at the phone itself.

**Fix**

```lua
ui = GetCurrentResourceName() .. '/ui/index.html',
```

The phone refuses the declaration rather than showing a blank page you cannot diagnose.

### A custom app opens on a blank page, with no error

**Cause:** the files are not declared in your own `fxmanifest.lua`. A file that is not
declared is not served to the player, and there is no message.

**Fix:** in **your** resource:

```lua
files {
    'ui/index.html',
    'ui/**',
}
```

The same applies to the bridge script: `custom_app_bridge/*.js` must be present in
`nash_phone`'s own `files` block, or
`https://cfx-nui-nash_phone/custom_app_bridge/nash-bridge.js` returns a 404 and your app stays
mute.

### A custom app's data never arrives

**Cause:** the app loads its data from `onUse`. The app is opened before your interface has
finished loading, so the message lands in the void.

**Fix:** have the interface ask, with `fetchNui`, once it is ready. Never push from `onUse`.

### `[nash_phone] app personnalisée « … » refusée : plafond de 40 atteint (Config.CustomAppsSettings.max)`

**Cause:** more than 40 custom apps registered. Usually a resource calling `AddCustomApp` in
a loop.

**Fix:** set `Config.CustomAppsSettings.debug = true` in `config/custom_apps.lua` to log every
add and remove, and find the caller. Raise `max` only once you know why you need to.

### A custom app's page loads unstyled, or its font is wrong

**Cause:** its stylesheet or font is pulled from an external URL. The phone ships every asset
it needs inside the resource: Inter is self-hosted as local `woff2` files rather than fetched
from Google Fonts: because the in-game browser is not a place to fetch stylesheets and fonts
at load time.

**Fix:** ship your CSS, fonts and icons inside your own resource and declare them in
`files{}`. Reference them as `https://cfx-nui-<your_resource>/…`. Remote **image** URLs do work
(that is how gallery photos hosted on Fivemanage display), but do not build a page that needs
a CDN to render.

The same reasoning explains `service_icons/`: it lives **outside** `web/build/` so that you can
drop a PNG in without touching the bundle, and because a rebuild empties `web/build/`.

## Configuration traps

### Editing `Config.Music.distance` in `config/main.lua` changes nothing

**Cause:** `config/music.lua` assigns `Config.Music` as a whole and is loaded **after**
`config/main.lua`. The audio block written in `main.lua` (`distance`, `defaultVolume`,
`maxVolume`, `stopOnDeath`) is therefore replaced and never read. The code falls back to its
own built-in values, which happen to match the shipped ones: so nothing looks wrong.

**Fix:** put the four audio keys in `config/music.lua`, alongside `tracks` and `playlists`:

```lua
Config.Music = {
    distance = 12.0,
    defaultVolume = 0.4,
    maxVolume = 1.0,
    stopOnDeath = true,
    tracks = { … },
    playlists = { … },
}
```

`/phonecheck` reports this as `reglages audio de Config.Music perdus : config/music.lua ecrase
config/main.lua.`

### Places in the Maps app land next to the right spot

**Cause:** the two spans of `Config.Map.Bounds` are not equal. Map tiles are square: one tile
covers as many metres across as it does down. If `maxX - minX` differs from `maxY - minY`, the
projection stretches one axis and places drift further the further they are from the centre.

**Fix:** keep both spans equal (shipped: 12000 × 12000). To recentre, move **both** bounds of
the same axis by the same amount so you do not reintroduce a stretch. Stand somewhere
recognisable, run `/phonepos`, and paste the printed block into `config/maps.lua`.
`/phonecheck` flags non-square bounds.

### No phone number is assigned, or numbers are truncated

**Cause:** `Config.Phone.prefix` and `Config.Phone.digits` produce a string longer than the
`VARCHAR(16)` column. The generated form is `prefix-digits`, so `555` + `7` gives
`555-XXXXXXX`, 11 characters. Depending on your SQL mode, the insert either fails or the
number is silently cut.

**Fix:** keep `#prefix + 1 + digits` at 16 or below. `/phonecheck` computes it and refuses to
call the install clean above that.

### `[nash_phone] Config.Earbuds is on but the item "phone_earbuds" does not exist in your inventory`

**Cause:** same as the phone item, in yellow because nothing vital breaks: only private
listening is unreachable.

**Fix:** create the item in your inventory (or import `sql/item_earbuds.sql` if your
framework keeps items in the database), or set `Config.Earbuds.enabled = false` in
`config/main.lua` to hide the feature entirely.

### Every service shows "Unavailable"

**Cause:** `Config.Services.ownDuty = false` and nothing is calling the duty export. With
`ownDuty = false`, the phone stops showing its own duty switch and waits for another script to
tell it who is on duty.

**Fix:** either put `ownDuty` back to `true`, or call the export from your clock-in system:

```lua
exports['nash_phone']:SetServiceDuty(source, true)
```

Also check that `job` in `config/services.lua` matches your framework's job name **exactly** -
it is the only thing the server looks at.

### A player is stuck on the byCloud screen during first-launch setup

**Cause:** the rate limiter. Sign-in and account creation share two counters: 8 failures on
one address, or 24 across all addresses, refuse everything for five minutes. That is what stops
the creation screen being used as a directory of the server's accounts. If
`Config.Setup.skippable` and `Config.Setup.cloudAccount.skippable` are **both** `false`, the
player can neither sign in, nor create, nor skip until the five minutes are up.

**Fix:** leave `cloudAccount.skippable = true` (the shipped value). An honest player never
reaches those numbers.

### The "Later" button never appears during setup

**Not a bug.** `Config.Setup.skippable = true` and `Config.Setup.passcode.required = true`
contradict each other: leaving without finishing means leaving without a passcode, which
`required` forbids, and the server would refuse to save. Rather than showing a button that
does nothing, the phone hides it. To actually get it: `skippable = true` **and**
`passcode.required = false`.

## Languages

### `[nash_phone] locales/zh.json est introuvable. Langue « zh » ignoree.`

**Cause:** the file name does not match the code in `Config.Locale` / `Config.ExtraLocales`,
or the file is not in `locales/`.

**Fix:** the file must be `locales/<code>.json` for the exact code you wrote. Also check that
`locales/*.json` is still declared in the `files` block of `fxmanifest.lua`: an undeclared file
is not sent to the client and `LoadResourceFile` returns nil there.

### `[nash_phone] locales/zh.json est illisible (JSON invalide). Langue ignoree.`

**Cause:** malformed JSON, most often a trailing comma on the last line.

**Fix:** validate the file. It is wrapped in a `pcall` on purpose: without it, one stray comma
in a buyer's file would fail the whole resource. The file must stay UTF-8.

On success the console confirms:

```
[nash_phone] Langue « zh » chargee : 2144 texte(s) d ecran, 34 message(s).
```

### Raw keys such as `settings.general` appear on screen

**Cause:** a string is missing from every language file. The fallback chain is the chosen
language, then English, then French, then the raw key.

**Fix:** the key on screen leads you straight to the line to fill in. This is a signal, not a
crash: a partial translation is usable, and an update that adds strings leaves them in English
until you catch up. Compare the new `locales/en.json` with yours to find what is missing.

## Events and integration

### `AddEventHandler` never fires

**Cause:** the wrong separator. Two prefixes are in use and they are **not** interchangeable:

| Prefix | Used for | Examples |
|---|---|---|
| `nash-phone:` (hyphen) | Phone domain events, meant for your scripts | `nash-phone:numberChanged`, `nash-phone:onAddTransaction`, `nash-phone:social:posted` |
| `nash_phone:` (underscore) | Resource-level plumbing | `nash_phone:event`, `nash_phone:open`, `nash_phone:requestOpen`, `nash_phone:toast` |

A handler on the wrong spelling is silent: no error, no call.

**Fix:** check the spelling against [Server Events](developer-api/server-events.md).

### `SendMessage` sends nothing

**Cause:** the **first** argument is the sender's phone *number*, not a `source`. Passing a
source silently produces a message from a number nobody owns, or is rejected outright when the
sender or body is empty.

**Fix**

```lua
local number = exports['nash_phone']:GetPhoneNumber(source)
exports['nash_phone']:SendMessage('555-0100', number, 'Your order is ready')
```

`SendNotification` and `EmergencyNotification` are the ones that accept **either** a source or
a number as their target.

### A player never receives anything until they open their phone

**Fixed in the current release**, and worth knowing if you are on an older build. Presence used
to be registered only inside the `bootstrap` callback, so a player who had not yet taken their
phone out was unreachable: no incoming SMS push, no notification row written, incoming calls
refused as "offline", no mail, no social message, invisible to AirDrop and to the Snapz map,
and `GetSourceFromNumber` returned nil. Presence and the state bags are now set when the
framework finishes loading the player.

Note this **changes** behaviour: players who used to be unreachable now receive, so rows are
written for them and calls that answered "offline" now ring.

## Money

### `[nash_phone] virement … : crédit ET remboursement impossibles, argent PERDU.`

**Cause:** the recipient disconnected between the balance check and the credit, and the
framework then also refused to refund the sender. The money is gone. This requires two
consecutive framework failures and should be rare.

**Fix:** this line names the sender, the recipient and the amount; refund by hand. If it
happens more than once, the framework's money functions are the thing to investigate, not the
phone: the debit and credit are chained with no yield between them precisely to keep that
window shut.

### `[nash_phone] virement … : argent transféré, historique NON enregistré (erreur SQL).`

**Cause:** the transfer went through, but the two `nash_phone_bank` rows failed to write. The
money movement is authoritative; the history is only its trace.

**Fix:** check the database is healthy (`/phonedeps` reports whether it actually answers: a
started `oxmysql` on a wrong connection string still reports `started`). The players' balances
are correct; only the Wallet history is missing those two lines.

## Miscellaneous

### `[nash_phone] réglages refusés (…) : taille sérialisée au-delà de la limite`

**Cause:** the settings blob for that identifier exceeds 8 192 serialized bytes. On a normal
install this cannot happen: it is a safety net against a modified client, and against future
keys turning the row into a dumping ground.

**Fix:** nothing on your side. Unknown keys are dropped anyway; only keys listed in `ALLOWED`
(`server/apps/settings.lua`) are ever written, and stray keys from an older version disappear
on the next write.

### "Test commands disabled. Set Config.Debug = true in config/main.lua."

**Cause:** a player ran `/phonehelp` or another `/phone*` command with `Config.Debug = false`.
The commands are registered even in production, so anyone can type them; this refusal is the
only line of `server/dev/` an ordinary player can see, and it is translated into their own
language.

**Fix:** expected behaviour. Enable `Config.Debug` only for testing, and put it back to
`false` before opening the server: those commands write to the database and push events in the
player's name.

### `/phonecheck` reports nothing wrong but nothing is saved

**Cause:** `oxmysql` is `started` on a wrong connection string. The resource state proves
nothing about the database.

**Fix:** `/phonedeps` runs a real `SELECT 1` and reports
`aucune reponse (verifie mysql_connection_string)` when it fails. Fix
`set mysql_connection_string` in `server.cfg` and restart.

## Related

- [FAQ](faq.md): the questions behind most of these fixes
- [First Launch](installation/first-launch.md): the checks to run before opening
- [Configuration Files](config/README.md): every key in `config/`
- [Commands](developer-api/commands.md): the full list of diagnostic and test commands
