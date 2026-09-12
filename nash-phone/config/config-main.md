# config/main.lua

The main configuration file. It is loaded **first**, and it is the file that creates the
`Config` table: every other file in `config/` adds to it.

File: `config/main.lua`

## Language

| Key | Type | Default | Description |
|---|---|---|---|
| `Config.Locale` | `string` | `'fr'` | Server language. Sets the language of out-of-phone notifications, and the starting language of a phone the player has never touched |
| `Config.ExtraLocales` | `table` | `{}` | Extra language codes offered in Settings › General › Language, on top of French and English |

`Config.Locale` accepts any code that has a matching file in `locales/`. The phone ships with
`locales/fr.json` and `locales/en.json`. To add one, copy `locales/en.json` to
`locales/<code>.json`, translate it, and write that code here: nothing to compile.

The value is normalised before use: case and surrounding spaces are tolerated (`'EN'`, `' en '`
both work). A code with **no matching file falls back to French**, not to English, and it does
so silently: if the phone stays in French after you changed this line, the file name is the
first thing to check.

A player who picks a language in Settings overrides this choice, and it is kept across
sessions.

`Config.ExtraLocales` is only needed when you want players to be able to **choose** between
several languages. To simply run the whole server in another language, `Config.Locale` alone is
enough: the server language is added to the list automatically.

```lua
Config.Locale = 'fr'
Config.ExtraLocales = {}          -- e.g. { 'zh', 'es' }
```

## Out-of-phone notifications

`Config.NotifyStyle`: what a player sees when a text message, a transfer or a missed call
arrives while the phone is **put away**.

| Value | Result |
|---|---|
| `'phone'` (default) | The phone's own banner, iOS style. The frame rises from wherever the player parked it, at the scale they chose |
| `'oxlib'` | A standard `ox_lib` toast, in your server's style |
| `'both'` | Both, mostly useful while comparing the two |

{% hint style="info" %}
`'oxlib'` does not remove the phone banner for everything. Notifications written by the server
(text message, mail, transfer, missed call) always reach the web page, which is never unloaded
and raises the frame on its own. What `Config.NotifyStyle` adds or removes on that path is the
`ox_lib` toast. The setting fully replaces the banner only for client-side notices: the ones
pushed by `PhoneNotify` and by the `Notify` export.
{% endhint %}

## Video calls (FaceTime)

During a video call, the phone's camera acts as a webcam: each player sends their own game
render to the other. **The image never passes through your server:** it travels player to
player, and the server only relays the few signalling messages.

```lua
Config.VideoCall = {
    enabled = true,
    debug = false,
    longEdgePx = 640,
    fps = 24,
    framing = 0.35,
    iceServers = {
        { urls = 'stun:stun.l.google.com:19302' },
    },
}
```

| Key | Type | Default | Accepted range | Description |
|---|---|---|---|---|
| `enabled` | `boolean` | `true` | - | `false` keeps video calls audio-only, with no image |
| `debug` | `boolean` | `false` | - | Writes each step of the call into the player's F8 console, from opening the camera to the connection state |
| `longEdgePx` | `number` | `640` | 160 to 1280 | Long edge of the stream sent. Above 640 the thumbnail gains nothing visible and bandwidth climbs |
| `fps` | `number` | `24` | 5 to 30 | Frames per second sent |
| `framing` | `number` | `0.35` | 0.05 to 0.95 | Where to aim inside the game image in selfie mode |
| `iceServers` | `table` | one Google STUN entry | - | Servers that help the two players find each other |

Values outside the accepted range are clamped, not refused.

**About `framing`.** In selfie mode GTA does not place the character in the middle of the
screen: they sit to the left. `0.5` is the centre of the screen, `0.30` clearly to the left,
`0.70` to the right. If your correspondent sees you too far to the **right** of their screen,
lower the value. The change applies to the next call, with nothing to rebuild.

**About `iceServers`.** A STUN server tells both players their public address; the free Google
one is enough in most cases. A **TURN** server *relays* the image when a direct connection is
impossible (players behind CGNAT, some 4G routers, corporate networks). Without TURN those
players get sound but no image: the rest of the call works normally. TURN is rented or
self-hosted (coturn); uncomment the second block and paste your credentials, keeping the UDP,
TCP and TLS entries, which is what gets through the most firewalls.

```lua
-- {
--     urls = {
--         'turn:turn.example.com:3478?transport=udp',
--         'turn:turn.example.com:3478?transport=tcp',
--         'turns:turn.example.com:5349?transport=tcp',
--     },
--     username = 'user',
--     credential = 'password',
-- },
```

An `iceServers` list that ends up empty falls back to the Google STUN entry, so the phone never
starts a call with no signalling at all.

## Brand shown inside the phone

The brand name that appears **inside** the phone's operating system. Put your own server's name
here; nothing is imposed.

```lua
Config.Brand = {
    name = 'NASH',
    studio = 'Nash',
    site = {
        label = 'NASH Store',
        url = 'nash-store.com',
        title = {
            fr = 'NASH Store : Scripts FiveM',
            en = 'NASH Store: FiveM Scripts',
        },
    },
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | `'NASH'` | Replaces the word visible to players: network operator in Settings › SIM, the account ID at the top of Settings, the device name in Settings › General › About, the Settings footer, and the sender of social-network verification codes |
| `studio` | `string` | `'Nash'` | Publisher shown on App Store pages. The suffix is added by the phone: `'Nash'` gives *Nash Arcade*, *Nash Labs*, *Nash Finance* |
| `site.label` | `string` | `'NASH Store'` | Short name under the Browser's bookmark icon |
| `site.url` | `string` | `'nash-store.com'` | Address displayed on that bookmark. No real navigation happens: the Browser is decorative |
| `site.title` | `table` | `fr` and `en` entries | Title listed under "Frequently visited", **one value per language**, keyed by language code |

Add one `site.title` line per language you declared in `Config.ExtraLocales` (`zh = '…'`,
`es = '…'`). A language with no line falls back to `label`; remove `title` entirely to show
`label` everywhere. Any field that is missing, empty or of the wrong type falls back to its
original value, so the phone never displays a blank.

## Mail

```lua
Config.Mail = {
    domain = 'ls-mail.com',
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `domain` | `string` | `'ls-mail.com'` | Domain of the phone's e-mail addresses |

A character's address is built the first time they open the Mail app, from their RP name:
`john.doe@ls-mail.com`. It is then **frozen**, because it is a destination: changing it would
break the mail of everyone who wrote it down. Two characters with the same name get a suffix
(`john.doe2@…`).

## byCloud account

A byCloud account **carries its owner's mailbox**. Signing in to it therefore gives access to
their mail; that is intended, and it is what this setting opens or closes.

```lua
Config.Cloud = {
    crossLogin = true,
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `crossLogin` | `boolean` | `true` | Can a player sign in to **another character's** byCloud account if they know the address and password? |

- `true` (shipped): yes. Stolen credentials have consequences: whoever gets in reads the
  owner's mail, sends from their address, and recovers their password keychain. This is a
  roleplay mechanic, not a vulnerability.
- `false`: no. An account can only be opened by the character who created it.

{% hint style="warning" %}
At `false`, a player no longer finds their account on another character. The account stops
being portable and becomes an annexe of one character. Weigh this if your players change lives
often.
{% endhint %}

The refusal is **silent, deliberately**: it answers exactly like a wrong password. Saying "this
account is not yours" would confirm that the account *exists*, and the sign-in screen would
become a directory of the server's accounts. A legitimate player never meets this refusal.

Accounts created before the update that introduced an owner have none recorded: they stay
openable by anyone, even at `false`. There is no way to guess who created them, and closing
them would lock their real owner out with no recourse.

## Test commands

```lua
Config.Debug = false
Config.DebugGroups = { 'admin', 'superadmin' }
```

| Key | Type | Default | Description |
|---|---|---|---|
| `Config.Debug` | `boolean` | `false` | Opens a set of commands that exercise the phone on its own: fake an incoming mail, a received message, a call, a story, a transfer. Type `/phonehelp` in game for the list |
| `Config.DebugGroups` | `table` | `{ 'admin', 'superadmin' }` | Framework groups allowed to run them once `Config.Debug` is on |

{% hint style="danger" %}
Leave `Config.Debug` at `false` in production. These commands write to the database and push
events in the player's name: open, they hand whoever finds them a way to grant themselves bank
transfers. Even when enabled they stay restricted to administrators, but the real barrier is
this boolean.
{% endhint %}

The server console always has access, since it already administers the server. The ACE
`nash_phone.debug` keeps working alongside the group list, for owners who prefer FiveM's native
permission system:

```cfg
add_ace identifier.license:XXX nash_phone.debug allow
```

An empty `Config.DebugGroups` means no group qualifies, and the ACE becomes the only path.

## Opening the phone

| Key | Type | Default | Description |
|---|---|---|---|
| `Config.OpenKey` | `string` | `'F1'` | Default key binding |
| `Config.OpenCommand` | `string` | `'phone'` | Chat command: `/phone` |

`Config.OpenKey` is registered through `RegisterKeyMapping`, so it is a **default**: each player
can remap it in FiveM's own key settings, and a binding a player has already stored wins over a
later change here. Setting `Config.OpenKey` to an empty string registers the command without any
key.

## Phone item

```lua
Config.UseItem = true
Config.ItemName = 'phone'
```

| Key | Type | Default | Description |
|---|---|---|---|
| `Config.UseItem` | `boolean` | `true` | The item must be in the player's pockets to open the phone |
| `Config.ItemName` | `string` | `'phone'` | Item name, to be created in **your** inventory |

The resource creates **no item:** your inventory declares items, not the phone. With
`Config.UseItem = true`, the item below must exist on your side and be in the player's pockets.

{% hint style="danger" %}
There is no escape hatch. `/phone` and the key binding go through the same server check
(`server/main.lua`, `requestOpen`). If the item does not exist, nobody opens the phone: not
players, not admins. The server prints a red warning in the console at startup when it detects
this case. Two ways out: create the item, or set `Config.UseItem = false`.
{% endhint %}

**ox_inventory:** add the entry to `ox_inventory/data/items.lua`:

```lua
['phone'] = {
    label = 'Phone',
    weight = 190,
    stack = false,      -- one phone per slot
    close = true,       -- closes the inventory when used
    description = 'A smartphone.',
    client = { image = 'phone.png' },
},
```

ox_inventory forwards item use to `ESX.RegisterUsableItem`, which the resource already
registers: nothing else to do. And if your version did not forward it, a player holding the
item still opens their phone with `/phone` or the key.

**ESX default inventory** (the `items` table): import the supplied `sql/item_phone.sql`, or
run:

```sql
INSERT INTO `items` (`name`, `label`, `weight`) VALUES ('phone', 'Phone', 1);
```

Then restart `es_extended`: ESX only reads the `items` table at startup.

**qb-inventory / QB base:** add the entry to `qb-core/shared/items.lua` under the same name as
`Config.ItemName`.

Handing the item out: `/giveitem <id> phone 1` (works on ESX and ox_inventory), or through your
shop or starter kit.

## Phone numbers

```lua
Config.Phone = {
    prefix = '555',
    digits = 7,
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `prefix` | `string` | `'555'` | Prefix of generated numbers |
| `digits` | `number` | `7` | Digits after the prefix: `555-XXXXXXX` |

Numbers are generated once per identifier, checked for uniqueness, and **never change hands**:
there is no `UPDATE` and no `DELETE` on the numbers table anywhere in the resource, not even in
the test commands.

{% hint style="warning" %}
The column is `VARCHAR(16)`. `prefix` + `-` + `digits` must stay at or under 16 characters,
otherwise, depending on your SQL mode, assigning a number either fails or silently truncates.
`/phonecheck` flags this before your players do.
{% endhint %}

## NearShare

Sharing a contact card with players **nearby**.

```lua
Config.NearShare = {
    radius = 15.0,
    maxPeers = 12,
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `radius` | `number` | `15.0` | Detection range around the player, in metres (GTA units) |
| `maxPeers` | `number` | `12` | Safety limit: how many players are listed at most |

The client scans and lists; the server re-checks the distance before accepting a transfer, with
a 5 m tolerance on top of `radius`. Without that margin, a player listed right at the edge would
be refused on send every other time.

## Notification retention

Dismissing a notification marks it **read:** it no longer comes back on the lock screen, but
the row stays in the database. Without a purge the table grows forever.

```lua
Config.Notifications = {
    keepReadDays = 3,
    keepDays = 30,
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `keepReadDays` | `number` | `3` | Days a notification already read is kept. `0` disables that sweep |
| `keepDays` | `number` | `30` | Days a notification is kept, read or not. `0` disables that sweep |

The sweep runs one minute after startup (to let the schema create itself on a first boot), then
every hour, across all players. With both values at `0` the loop exits and never runs again.

This is the **table-wide** cleanup. Clearing **one player's** notifications when they join or
leave is a different file: [config/notifications.lua](config-notifications.md).

## Prop and frame colour

```lua
Config.Prop = {
    model = 'custom_phone_prop',
    fallback = 'prop_npc_phone_02',
    bone = 28422,
    offset = vec3(0.0, 0.0, 0.0),
    rotation = vec3(0.0, 0.0, 0.0),
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `model` | `string` | `'custom_phone_prop'` | Prop model, streamed by the `nash-phoneprop` resource |
| `fallback` | `string` | `'prop_npc_phone_02'` | GTA prop used when the model above cannot be loaded |
| `bone` | `number` | `28422` | Bone index: right hand (`IK_R_Hand`) |
| `offset` | `vec3` | `vec3(0.0, 0.0, 0.0)` | Attachment offset |
| `rotation` | `vec3` | `vec3(0.0, 0.0, 0.0)` | Attachment rotation |

If the prop resource is not started, the model does not exist and the phone would be invisible
in the hand, so the fallback takes over: an approximate phone beats an empty hand. `offset` and
`rotation` are both zero and look removable: they are not. Deleting them raises an error out of
the attach path, and the player ends up with a full-screen phone but no control loop, free to
shoot and drive.

### Coloured shell

The prop's frame follows the colour picked in Settings › Display › Frame colour, with no
restart. The model is not duplicated: the three textures that carry the colour are swapped in
memory inside the prop's dictionary. One `.ydr`, no extra streamed asset.

```lua
skin = {
    enabled = true,
    txd = 'custom_phone_prop',
    textures = { body = 'Warm_Terracotta', light = 'Warm_Terracotta_light', back = 'back_iphone' },
    colors = {
        deepblue = { body = '#2c3144', light = '#3a3a43', back = 'assets/prop/back_deepblue.png' },
        cosmic   = { body = '#dc7a3b', light = '#d8bd9a', back = 'assets/prop/back_cosmic.png' },
        silver   = { body = '#d8dbdf', light = '#e6e9ed', back = 'assets/prop/back_silver.png' },
        black    = { body = '#2e3134', light = '#3a3d41', back = 'assets/prop/back_black.png' },
    },
    default = 'cosmic',
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | `false` leaves the prop on its original texture |
| `txd` | `string` | `'custom_phone_prop'` | Texture dictionary, as it exists in the prop's `.ytd` |
| `textures` | `table` | see above | Names of the three textures replaced |
| `colors` | `table` | 4 entries | One set per colour offered in Settings |
| `default` | `string` | `'cosmic'` | Colour applied before the player picks one |

`body` and `light` are flat colours built in memory; `back` is the camera block, a real image.

{% hint style="warning" %}
The colour swap is **local to each player**. On their screen every phone takes their colour,
including other players'. That is the limit of the method, and it is accepted as such.
{% endhint %}

Three lists must agree: the colour identifiers here, the swatches in the phone's Settings, and
the server's own whitelist (`frameColor`, currently `deepblue`, `cosmic`, `silver`, `black`). If
an identifier is missing from one of them, nothing errors: the player taps a swatch and either
nothing happens at all, or the on-screen frame changes colour while the prop held in hand stays
on the old one.

**The pair rule:** `light` must always be **lighter** than `body`. It is the flat colour of the
faces catching the light; if it drops below its `body`, the area meant to brighten darkens
instead and the frame's volume reads inverted. Nothing breaks and nothing warns: the prop just
looks dull.

The `back` images are recoloured **offline**. Changing a `body` here does not change the
matching back: the PNG has to be regenerated.

## Voice and speaker

```lua
Config.Voice = {
    provider = 'pma-voice',
    Speaker = {
        enabled = true,
        distance = 8.0,
        refreshSeconds = 2,
        maxListeners = 8,
    },
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `provider` | `string` | `'pma-voice'` | Voice resource used during calls |

{% hint style="warning" %}
`pma-voice` is the only provider with an implementation. Any other value leaves the voice layer
a no-op: the call rings, the call screen works and the signalling goes through, but no voice
channel is ever joined and the two players do not hear each other.
{% endhint %}

### Speaker

The **Speaker** button during a call brings the people **around the player** into the
conversation, like a real phone laid on a table: those in range hear the correspondent, and the
correspondent hears them.

It works **both ways**, and that is not a side effect: that is what a speaker is. If you do not
want it on your server, turn the feature off rather than trying to mute one direction.

The player who turns it on sees their button light up. The people around get **no on-screen
indication**: they simply start hearing the conversation. That is intended, and it is what makes
it a roleplay tool: you can listen in without anyone knowing, and you can be caught doing it.
Nobody is ever pulled into a call they cannot leave: walking away is enough. A player **already
in a call** is never captured by someone else's speaker.

| Key | Type | Default | Minimum | Description |
|---|---|---|---|---|
| `enabled` | `boolean` | `true` | - | `false` leaves the button visible but greyed out and inert |
| `distance` | `number` | `8.0` | `1.0` | Range in metres around whoever turned it on |
| `refreshSeconds` | `number` | `2` | `1` | How often the server re-checks who entered or left the range |
| `maxListeners` | `number` | `8` | `1` | Load ceiling on the number of listeners; the closest are kept |

`8` metres is roughly a room, or the length of a car. Below `4` you have to be practically
glued; above `15` you hear from across the street and the speaker becomes an ambient microphone.
Distance is measured continuously: someone walking up joins, someone walking off drops out.

The re-check only runs **while a speaker is on:** a server where nobody uses it pays nothing.
Two seconds is barely noticeable at walking speed; dropping to `1` doubles the work for a gain
the ear cannot hear.

`maxListeners` is a load limit, not a privacy setting. In the middle of a crowd, without it,
thirty people would join the same voice channel at once. Candidates are sorted by distance
before the ceiling is applied, so the ones kept are the closest, not whoever happened to come
first in the player list.

## Music (proximity audio)

{% hint style="danger" %}
This block is **dead as shipped**. `config/music.lua` assigns `Config.Music` as a whole and is
loaded after `config/main.lua`, so these four values are replaced and never read. The hard-coded
fallbacks below apply instead: they happen to be the same numbers, which is why nothing looks
broken until you change one and nothing happens. To actually change them, add them to the
`Config.Music` table in [config/music.lua](config-music.md). `/phonecheck` reports this.
{% endhint %}

```lua
Config.Music = {
    distance = 12.0,
    defaultVolume = 0.4,
    maxVolume = 1.0,
    stopOnDeath = true,
}
```

| Key | Type | Value in `main.lua` | Fallback used at runtime | Description |
|---|---|---|---|---|
| `distance` | `number` | `12.0` | `12.0` | Listening range in metres (proximity audio) |
| `defaultVolume` | `number` | `0.4` | `0.4` | Starting volume, `0.0` to `1.0` |
| `maxVolume` | `number` | `1.0` | `1.0` | Ceiling the player can reach |
| `stopOnDeath` | `boolean` | `true` | on | Cut the music when the player dies |

Pasting a link into the Music app plays it as **proximity audio**: players around hear it, like
a real phone speaker. This requires the **xsound** resource to be running. Without it the
feature is simply disabled, the player is told, and the rest of the phone works normally.

`stopOnDeath` is recommended: otherwise a body on the ground keeps broadcasting to the whole
block with nobody able to stop it. The player always keeps a Stop button (Music app and control
centre); closing the phone does **not** stop the music, which is intended.

## Earbuds (private listening)

By default, music from the Music app comes out of the **speaker** and everyone nearby hears it.
With earbuds, the player switches to private listening: they alone hear anything.

```lua
Config.Earbuds = {
    enabled = true,
    itemName = 'phone_earbuds',
    requireItem = true,
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | `false` removes the feature entirely and hides the setting |
| `itemName` | `string` | `'phone_earbuds'` | Item name, to be created in **your** inventory |
| `requireItem` | `boolean` | `true` | Must the earbuds be carried for them to work? |

How it plays out: the player owns the `itemName` item, **uses** it once to pair the earbuds with
the phone, then uses it again to connect or disconnect them like a real pair. The setting stays
reachable under Settings › Audio output. The item is never consumed.

At `requireItem = true`, losing or selling the earbuds sends music back to the speaker: the
item keeps a value, which is the realistic outcome. At `false`, once paired they work forever,
even without the item.

The resource creates no item here either; see `sql/item_earbuds.sql` for the default ESX
inventory. Without xsound there is no music to route, so the setting stays visible but has
nothing to do.

## Camera

Photos and videos are captured from the game view inside the phone, then sent to the host
configured in [config/upload.lua](config-upload.md). Uploads go through a presigned address:
**the API key never leaves the server**.

```lua
Config.Camera = {
    Photo = {
        LongEdgePx = 1920,
        Quality = 0.8,
    },
    Recording = {
        MaxDurationSeconds = 30,
        Fps = 30,
        Bitrate = 1200000,
        LongEdgePx = 720,
        Microphone = true,
    },
}
```

### Photo

| Key | Type | Default | Description |
|---|---|---|---|
| `LongEdgePx` | `number` | `1920` | Long edge of the picture file, in pixels |
| `Quality` | `number` | `0.8` | WebP quality, `0.0` to `1.0` |

The aspect ratio comes from the mode chosen in the app (3:4 in photo, 4:3 in landscape) and
`LongEdgePx` sets the long side: `1920` gives 1440×1920 in photo and 1920×1440 in landscape,
`2560` gives 1920×2560 and 2560×1920. Above 2560 mostly the file gets heavier.

### Recording

| Key | Type | Default | Accepted range | Description |
|---|---|---|---|---|
| `MaxDurationSeconds` | `number` | `30` | - | Maximum length of a video; recording stops on its own |
| `Fps` | `number` | `30` | - | Frames per second captured |
| `Bitrate` | `number` | `1200000` | - | Video bitrate in bit/s. Roughly 150 KB per second of film |
| `LongEdgePx` | `number` | `720` | 240 to 1080 | Long edge of the **video**, in pixels. The ratio comes from the mode (3:4), so `720` means 540×720 |
| `Microphone` | `boolean` | `true` | - | Record the voice of the player filming |

{% hint style="warning" %}
`Recording.LongEdgePx` is the setting that weighs most on in-game performance: the video is
encoded live while GTA V runs, and the cost follows the pixel count. Going to `1080` (810×1080)
multiplies that cost by 2.25 without a sharper image, because `Bitrate` stays the same and is
spread over more pixels. Dropping to `480` (360×480) lightens it further, for heavily loaded
servers.
{% endhint %}

The player's **microphone** can be recorded onto the video, like a vlog. The **game's** sound
cannot: the embedded browser never receives GTA's audio mix. `Microphone = false` removes the
option entirely and the microphone button disappears from the app; either way the player can
mute before each take.

## Adding photos by link

Players can add a picture to Photos by pasting a link, with the **+** button of the Albums tab
(Collections in French). The picture is downloaded **by the player's game**, checked,
re-encoded and uploaded to the host configured in [config/upload.lua](config-upload.md), the
same way as a camera photo. The server never downloads the pasted link: it only checks where
the final file is stored, then saves the row. A Discord link is therefore never stored as it
is, which matters because it would expire after 24 hours.

```lua
Config.Gallery = {
    Import = {
        Enabled = true,
        MaxMegabytes = 15,
        MaxSidePx = 8192,
        AllowedHosts = { 'fivemanage.com' },
    },
}
```

| Key | Type | Default | Accepted range | Description |
|---|---|---|---|---|
| `Import.Enabled` | `boolean` | `true` | - | `false` hides the **+** button, and the server refuses imports. Copy, Duplicate and the Recently Saved collection are not affected |
| `Import.MaxMegabytes` | `number` | `15` | 1 to 50 | Heaviest original file accepted, in megabytes. The player's error message quotes this number |
| `Import.MaxSidePx` | `number` | `8192` | 1024 to 16384 | Largest width or height of the original accepted, in pixels. Read from the file header, before the picture is decoded |
| `Import.AllowedHosts` | `table` | `{ 'fivemanage.com' }` | - | Hosts where the server agrees to **store** an imported picture. Each entry also covers its subdomains, so `fivemanage.com` covers `r2.fivemanage.com`. The address must be `https` |

Values outside the accepted range are brought back to the nearest bound, not refused: a
`MaxMegabytes = 0` left by mistake gives 1 MB, not a feature that silently refuses every
picture. A value that is not a number falls back to the default.

If your `config/main.lua` comes from an older version and has no `Config.Gallery` block at all,
adding by link is **on**, with the defaults above. Paste the block in to change them.

The **+** button only appears when `Enabled` is `true` **and** an image host is configured in
`config/upload.lua`: without a host there is nowhere to put the file.

The picture itself follows the [Camera](#camera) settings: saved as WebP, long edge at most
`Camera.Photo.LongEdgePx`, quality `Camera.Photo.Quality`. A smaller picture is not enlarged.
Re-encoding removes the EXIF metadata (GPS position included), and an animated GIF keeps only
its first frame. Accepted formats are JPEG, PNG, GIF and WebP, recognised from the content of
the file rather than from its name.

Each import costs up to two uploads (the picture and its thumbnail) out of the 12 per minute a
player is allowed: it is the same counter as the Camera. A link that is already on Fivemanage is
re-hosted like any other: Fivemanage serves every account's files from the same addresses, so
the phone cannot tell your files from another server's.

**About `MaxSidePx`.** A file can be light in bytes and enormous in pixels. Decoding it to
re-encode it takes 4 bytes per pixel in memory, in a browser that shares that memory with the
game. That is why the dimensions are read from the file header first, and a picture over the
limit is refused before anything is decoded. On top of this setting, the phone refuses any
picture above **50 million pixels** in total (about 200 MB once decoded): 8192×6000 goes
through, 8192×8192 does not. Raising `MaxSidePx` gains nothing visible, since every picture is
brought down to `LongEdgePx` anyway.

**About `MaxMegabytes`.** The download runs on the player's own connection and in the game's
memory, and gives up after 20 seconds. 15 MB is comfortably above a typical photo shared on
Discord.

### `AllowedHosts` and a custom Fivemanage domain

`AllowedHosts` lists where imported files may be **stored**, not where players may copy links
from. Any link the player's game can download can be imported, Discord included, because the
file is always re-hosted before it is saved.

The default covers Fivemanage's own addresses (`r2.fivemanage.com` and any other subdomain of
`fivemanage.com`). If your Fivemanage account serves your files from a **custom domain**,
uploads come back with addresses on that domain, and the server refuses them until you list it:

```lua
AllowedHosts = { 'fivemanage.com', 'media.yourserver.com' },
```

Without that line, every import fails at the very last step, after the file was uploaded. The
player gets "Something went wrong. Try again.", the uploaded file stays unused in your
Fivemanage storage, and the Camera keeps working, because it does not go through this check.
That combination is the tell-tale sign.

**How entries are read.** Write host names only. What an owner naturally types is tolerated and
cleaned up: capital letters, a leading `https://`, a trailing path or slash, a leading `*.`. An
entry with no dot at all (`'com'`) is ignored, since it would open a whole top-level domain, and
the server console lists every ignored entry at startup. A list that ends up empty falls back to
`fivemanage.com`, with a warning: to turn the feature off, use `Enabled = false`, not an empty
list. The stored address itself may not carry a port or an `@`.

{% hint style="warning" %}
List only hosts that serve **your** files. On a host listed here, a modified client can save any
address directly and skip every check the phone makes (format, size, re-encoding). Never add
Discord: its links expire after 24 hours, and `nashphone_cleanup` deletes them.
{% endhint %}
