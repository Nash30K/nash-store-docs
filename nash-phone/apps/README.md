# Application Catalogue

NASH Phone ships 32 applications, plus any custom app you add yourself. This page lists every one of them, states whether it is backed by the server and the database or whether it is a visual mockup, and names what has to be configured before it is of any use on your server.

{% hint style="warning" %}
Some stock apps are decorative on purpose and will never do anything: **Browser** (`safari`), **Weather** (`weather`), **Health** (`health`), **Stocks** (`stocks`), and the "Lives" tab of **TickTok**. **PlayTube** has a real account system, but its videos, channels, comments and private messages never leave the player's own phone. The full recap is in [Decorative apps](#decorative-apps-on-purpose) at the bottom of this page. Read it before you promise anything to your players.
{% endhint %}

## Status legend

Every table on this page uses the same four statuses.

| Status | Meaning |
| ------ | ------- |
| **Networked** | Backed by `server/apps/*.lua` and the database. Content crosses from one player's phone to another and survives a server restart. |
| **Personal** | Fully functional, but private to the player. Nothing crosses between players. The "Stored in" column says whether the state is saved server-side per character or only in the phone's local storage. |
| **Live game data** | Reads the running game (position, heading, in-game time). No database, nothing shared, nothing to configure beyond the values already in `config/`. |
| **Mockup** | Fixed or locally generated content. Nothing a player does inside it reaches another player, and most of it resets when the phone reloads. |

"Personal (local)" is the one worth flagging to your players: it lives in the CEF browser storage of the player's own game client. It survives a NUI reload and a reconnect on the same machine, but not a cache wipe or a different computer. NASH Phone clears those keys when the character behind the phone changes, so a second character never inherits the first one's data.

## Enabling, disabling and renaming apps

Everything on this page is controlled from `config/apps.lua`. Three optional keys per app:

| Key | Type | Default | Effect |
| --- | ---- | ------- | ------ |
| `enabled` | `boolean` | `true` | `false` removes the app completely: no icon, absent from the App Store, the app library, search and Settings. It is also removed from the home screen of players who already had it. |
| `name` | `string` | none | Display name shown instead of the built-in one. It **replaces the translation**, so the same name is shown in French and in English. That is intended: "Chicago Taxi" does not need translating. |
| `store` | `boolean` | per app | `true` = the app is not on the phone at first start, the player has to download it from the App Store. `false` = preinstalled on the home screen. |

An app you leave out of the table keeps its original values, so you can delete the lines you do not care about. The server only sends the *differences* to the phone at bootstrap, which is why an app added by a later update still works even if it is missing from your file.

### Default install state

| Where | Apps |
| ----- | ---- |
| Dock (always in reach) | `phone`, `messages`, `camera`, `wallet` |
| Preinstalled on the home screen | `contacts`, `gallery`, `mail`, `notes`, `maps`, `weather`, `clock`, `calendar`, `facetime`, `music`, `safari`, `health`, `reminders`, `passwords`, `services`, `appstore`, `settings` |
| App Store download (`store = true`) | `calculator`, `compass`, `stocks`, `snake`, `tictactoe`, `ticktok`, `instapic`, `playtube`, `snapz`, `chatapp`, `birdby` |

{% hint style="danger" %}
Disabling `appstore` makes every app marked `store = true` unreachable for the player: the App Store is the only in-phone install path. The apps are not deleted, they simply cannot be installed any more. The only way back in is the client export `exports.nash_phone:SetAppInstalled(app, true)` called from another resource.
{% endhint %}

Two more apps carry the same kind of weight. Disabling `phone` removes calls and the call log; disabling `messages` removes SMS, and other scripts can still send them but the player will never see them. Disabling `settings` leaves the player unable to change wallpaper, ringtone, language or account.

## Communication

| App | `id` | Status | Stored in | Server file |
| --- | ---- | ------ | --------- | ----------- |
| Phone | `phone` | Networked | `nash_phone_calls` | `server/apps/phone.lua` |
| Messages | `messages` | Networked | `nash_phone_messages` | `server/apps/messages.lua` |
| Contacts | `contacts` | Personal (server) | `nash_phone_contacts`, `nash_phone_appdata` | `server/apps/contacts.lua` |
| Mail | `mail` | Networked | `nash_phone_mails`, `nash_phone_mail_accounts` | `server/apps/mail.lua` |
| VideoCall | `facetime` | Networked | `nash_phone_calls` | `server/apps/phone.lua`, `server/apps/webrtc.lua` |
| Services | `services` | Networked | `nash_phone_service_requests` | `server/apps/services.lua` |

<details>

<summary>Phone</summary>

Keypad, favourites, recents and a contacts tab. Places real calls to another player's number.

**Real.** Call signalling is authoritative and lives in server memory; every finished or missed call is written to `nash_phone_calls` with an `app_id` column. The Phone app shows **every** call whatever app placed it, and labels the origin on the line, the way a FaceTime call also appears in Phone on iOS. VideoCall and ChatApp filter the same log down to their own `app_id`.

**Voice.** Each call gets a dedicated voice channel (channel numbers start at 50000, above the usual radio range) driven through `Config.Voice.provider` in `config/main.lua`, `'pma-voice'` by default. Without a voice resource running, the call screen still works and the log is still written, but nobody hears anybody.

**Conference.** A call keeps its original pair and accepts guests. There is no host: anybody can invite, anybody can leave, and the call ends when one person is left. Hard cap of **4 participants** including guests (`MAX_PARTS`), because video travels as a full mesh: at four, each player already holds three connections and sends three copies of their stream.

**Speaker.** The speaker button pulls the people *around* the player into the conversation, in both directions, as a real speakerphone would. Configured in `Config.Voice.Speaker` (`config/main.lua`): `enabled = true`, `distance = 8.0` metres, `refreshSeconds = 2`. Bystanders get no on-screen indication that they are being heard, which is the point. A player already in a call is never pulled into someone else's speaker.

**Limits.** An unanswered call is marked missed after 30 seconds. Calls are not persisted as sessions: a server restart ends every call in progress.

</details>

<details>

<summary>Messages</summary>

SMS between players, with threads derived from the message table and real-time delivery.

**Real.** Three message types are accepted and nothing else: `text`, `image`, `location`. The whitelist is enforced server-side because the payload comes from the NUI, and an unknown type would render as an empty bubble on both phones.

**Configuration.** None required. Other resources can push messages through `exports.nash_phone:SendMessage(from, to, message)` and a position through `exports.nash_phone:SendCoords(from, to, coords)`.

**Notes.** A shared position is never rendered as its raw payload in previews or banners; the preview text is localised per recipient, using each player's own phone language rather than the server language.

</details>

<details>

<summary>Contacts</summary>

The address book: create, edit, delete, mark as favourite, attach a photo.

**Real, but private.** Contacts are per character in `nash_phone_contacts`. Contact photos are stored separately, as a JSON block in `nash_phone_appdata` under the `contacts` app key, because they only ever hold **image addresses, never binary data**.

**AirDrop.** A contact card can be sent to nearby players. The client scans for players inside `Config.NearShare.radius` (default `15.0` metres, `maxPeers = 12`) and the server re-checks the distance before delivering, so a modified client cannot AirDrop across the map.

**Why it matters beyond Contacts.** ChatApp has no directory of its own: its conversation list is deduced from this address book. A contact added here shows up in ChatApp on its own, if that person has a ChatApp account.

**Limits.** `display_name` is capped at 64 characters, matching the column. That cap exists because an AirDropped card is written by the *sender* into the *recipient's* address book.

</details>

<details>

<summary>Mail</summary>

A real mailbox: inbox, sent, archive, compose, and mail that actually travels from one player to another.

**Real.** One row per mailbox: sending writes two rows, one in the recipient's `inbox` and one in the sender's `sent`. Each side archives, reads and deletes their own copy without ever touching the other's.

**Addresses.** A character's address is built from their RP name the first time they open the app (`john.doe@ls-mail.com`) and then **frozen**, because it is a destination. Two characters with the same name get a numeric suffix. The domain comes from `Config.Mail.domain` in `config/main.lua`, default `'ls-mail.com'`.

**The byCloud crossover, which is a feature.** A byCloud account belongs to the character who created it and *carries their mailbox*. Signing into someone else's byCloud account therefore opens their mail. This is deliberate roleplay design, not a leak: stealing credentials is meant to have consequences. Close it with `Config.Cloud.crossLogin = false` in `config/main.lua`, at the cost of the account no longer being portable between characters.

**Limits.** In game the mailbox starts empty; there is no seeded welcome mail. Mail is the only one of the four "personal" apps that crosses between players, which is why it has its own tables instead of living in `nash_phone_appdata`.

</details>

<details>

<summary>VideoCall (`facetime`)</summary>

Video calls between players. Same call system as the Phone app, flagged as video.

**Real.** The picture is the caller's **game view**, not a webcam: the game render is painted into a canvas by the CEF game-view plugin, turned into a stream, and sent peer to peer. Your server never sees a frame; it only relays the handshake (`server/apps/webrtc.lua`).

**Configuration** in `Config.VideoCall` (`config/main.lua`):

| Key | Default | Meaning |
| --- | ------- | ------- |
| `enabled` | `true` | `false` keeps calls audio-only, with no picture |
| `longEdgePx` | `640` | long edge of the transmitted stream |
| `fps` | `24` | frames per second sent |
| `framing` | `0.35` | where to aim in the game image in selfie mode; lower it if your correspondent sees you too far right |
| `debug` | `false` | writes the whole call sequence to the player's F8 console |
| `iceServers` | Google STUN | STUN and, optionally, TURN |

{% hint style="info" %}
Without a TURN server, players behind CGNAT, some 4G boxes and corporate networks get the **audio but no picture**. The rest of the call works normally. A commented TURN block is provided in `config/main.lua`; keep UDP, TCP and TLS entries, they pass the most firewalls.
{% endhint %}

</details>

<details>

<summary>Services</summary>

Citizens contact the police, EMS, a mechanic or a taxi; employees of those jobs answer from a "My service" page.

**Real.** Duty state is kept in server memory (it is a session state: a restarted server has nobody online, therefore nobody on duty). Requests are written to `nash_phone_service_requests` so an employee finds their log after a restart. The NUI never declares which job the player holds or whether they are on duty: the server reads it from the framework.

**Configuration** in `config/services.lua`. The order of `list` is the displayed order.

| Key | Default | Meaning |
| --- | ------- | ------- |
| `enabled` | `true` | master switch |
| `requestTimeout` | `15` | minutes before an unanswered request expires |
| `historyKeep` | `25` | requests kept in a company's "recent requests" log |
| `cooldown` | `20` | seconds between two requests from the same player, `0` disables |
| `ownDuty` | `true` | the app's own duty switch is the truth. `false` hides the switch and waits for another script to call `exports.nash_phone:SetServiceDuty(source, on, receive)` |

Per company: `job` (must match your framework's job name exactly), `label`, `place`, `icon` (a PNG dropped in the resource's `service_icons/` folder, nothing to rebuild), `accent`, `canCall`, `canMessage`, `multiResponders`, `shareLocation`.

{% hint style="warning" %}
With `ownDuty = false` and nothing calling `SetServiceDuty`, **every service stays "Unavailable"**. That is not a bug: no employee has ever been declared on duty. `exports.nash_phone:GetServiceOnDuty(job)` reads the current list back.
{% endhint %}

A company absent from `list` does not exist in the app, and a player holding that job will not see a "My service" page either.

</details>

## Social apps

The six social apps share one foundation, so read this before the per-app blocks.

**Accounts are real.** Every social app has its own account table row in `nash_phone_social_accounts`, with a salted SHA-256 password (hashed by MySQL, no Lua crypto dependency). Sign-up, sign-in and password reset by verification code all work. Sign-in on one app has no effect on another.

**There is no account directory, and there must never be one.** No endpoint lists accounts. You search for a username (minimum 3 characters, maximum 10 results, private accounts excluded) or you do not find anybody. On a 64-slot server, a browsable list would be the full roster of who is playing and how many, which is metagaming served on a plate. ChatApp is the exception, and it works the other way round: it has no search at all, its conversation list is deduced from the phone's address book.

**Addressing.** ChatApp addresses people by phone **number**, like WhatsApp. Snapz, InstaPic, BirdBy and TickTok address by **username**.

**Shared tables.** Posts, likes, comments, follows and in-app notifications live in one generic set of tables carrying an `app_id` column: `nash_phone_social_posts`, `_likes`, `_comments`, `_follows`, `_notifs`. Private messages live in `nash_phone_social_messages`, stories in `nash_phone_social_stories` and `nash_phone_social_story_views`.

**Feeds only show what you follow.** No app returns "the server's posts" or "popular accounts". Same reason as above.

**Media needs an image host.** Every photo posted in a social app goes through the same upload path as the Camera, so `config/upload.lua` must be filled in. See [Camera](#media-and-capture).

| App | `id` | Feed / posts | Private messages | Stories | Extras |
| --- | ---- | ------------ | ---------------- | ------- | ------ |
| ChatApp | `chatapp` | Status feed from stories | Real, by number | Real | Calls logged under `chatapp` |
| InstaPic | `instapic` | Real | Real, by username | Real | **Live broadcasts** (real) |
| Snapz | `snapz` | Real | Real, by username, incl. snaps | Real | **Map with real in-game positions** |
| BirdBy | `birdby` | Real | Real, by username | none | Replies and follows real |
| TickTok | `ticktok` | Real | Real, by username | none | "Lives" tab is a **mockup** |
| PlayTube | `playtube` | **Mockup** | **Mockup** | none | Real account, real YouTube link player |

Shared server limits, enforced server-side: post body 500 characters, place 140, comment 500, feed page 20; private message body 1000 characters, page 40; story text 280 characters, visible for 24 hours, **40 live stories maximum per account**, 60 authors and 600 rows scanned per carousel read.

<details>

<summary>ChatApp</summary>

WhatsApp-style messenger: Status, Calls, Chats and a "You" tab.

**Real.** Conversations, messages and status posts all cross between players. Addressing is by phone number.

**No directory, by design.** ChatApp has no search and no friend requests. Its contact list is deduced from the phone's address book: add someone in the Contacts app and they appear here on their own, if they have a ChatApp account. A stranger never appears.

**Calls.** Calls placed from ChatApp use the same call system as the Phone app and are logged with `app_id = 'chatapp'`, so ChatApp keeps its own call history.

**Avatars.** The profile picture is captured from the player's RP character head in game (via `screenshot-basic`), then uploaded to the configured image host.

</details>

<details>

<summary>InstaPic</summary>

Photo network: feed, stories, direct messages, activity, profiles, and live broadcasts.

**Real, all of it.** The feed, likes, comments, follows, the activity tab and stories are server-backed and shared. Direct messages are addressed by username. What is left on the phone is only the appearance the player picks for themselves (avatar gradient and emoji) and their saved posts.

**Stories audience** is set by `Config.Social.StoriesAudience.instapic` in `config/social.lua`, shipped as `'everyone'`: every story posted on the server in the last 24 hours appears in the carousel, even from accounts the player does not follow. Set `'following'` to restrict it to accounts the player follows plus their own. A private account only ever shows its stories to its followers, whatever this setting says.

### Live broadcasts

A live broadcasts the host's **real game view** to viewers. It is the same transport as a video call: the picture is painted into a canvas, turned into a stream, and sent directly from player to player. **The image never passes through your server**, which only relays the handshake.

Configured in `Config.Social.Live` (`config/social.lua`):

| Key | Default | Meaning |
| --- | ------- | ------- |
| `enabled` | `true` | `false` keeps the button visible but the server refuses to start a broadcast, and the app shows a clear refusal instead of a dead button |
| `maxViewers` | `8` | viewers per live, clamped server-side to 1..24 |
| `defaultAudience` | `'followers'` | who gets notified at launch: `'everyone'`, `'followers'` or `'none'`. The host can change it at launch time |

{% hint style="warning" %}
`maxViewers` is a physical limit, not a comfort setting. With no media server, the host sends a **separate stream to every viewer**: their upload and CPU cost rise proportionally. At 8, a player on fibre holds up. Past a dozen, the broadcast degrades for everyone already watching, and the host loses frames in their own game. Raising the number does not fit more people in, it splits the same capacity across more of them.
{% endhint %}

Other fixed limits: the host plus **3 guests** may broadcast; title 60 characters; comment 200 characters; the last 60 chat lines are kept for late arrivals. The list of active lives only returns broadcasts from accounts the requester already follows, plus their own.

**No SQL table, on purpose.** A live only makes sense while its host is connected. An interrupted live is a finished live; nothing is restored after a restart. Launch notifications stay *inside* the phone and never draw over the game screen, unlike an incoming call.

Lives use `Config.VideoCall.iceServers`, so the TURN caveat from VideoCall applies here too.

</details>

<details>

<summary>Snapz</summary>

Snapchat-style app: chats, snaps, stories and a friend map.

**Real.** Conversations, snaps and stories cross between players. Addressing is by username. A **snap opens only once**: the server erases the media in the same write as the opening, so a second attempt cannot return anything.

**Stories audience** is `Config.Social.StoriesAudience.snapz`, shipped as `'following'` (unlike InstaPic), matching the app it is modelled on.

### The map

The map shows where friends **really are in the game world**. It is the most sensitive setting in `config/social.lua`, because an in-game position is in-game information. Four locks, none of which a player can bypass:

1. **Both follow each other.** A one-way follow is not enough, so nobody can follow someone just to track them.
2. **The person said yes.** The map asks on first open and shows nothing until answered. Going ghost again is one gesture, at any time.
3. **They are connected.** A position is never stored in the database; it is read from the game when the map is opened and disappears with the disconnect. Nobody can find out where someone stopped playing.
4. **You left the map enabled** in the config.

| Key | Default | Meaning |
| --- | ------- | ------- |
| `enabled` | `true` | `false` keeps the tab but shows nobody, and tells the player so instead of leaving them on an empty map |
| `requireMutual` | `true` | lock 1. `false` means following someone is enough to see them, without their knowledge |
| `refreshSeconds` | `5` | polling rate while the tab is open only. Below 3 seconds nothing visible is gained |
| `blurMeters` | `0` | `0` = exact position. Above zero, positions are rounded onto a grid of that size, applied **by the server**, so a modified client gains nothing. 150 m is roughly a city block |
| `maxFriends` | `50` | load guard, not a privacy setting |

The map can only ever show people the player already follows mutually, so people they have met in game.

</details>

<details>

<summary>BirdBy</summary>

Twitter-style app: timeline, replies, likes, follows, alerts, private messages, profiles.

**Real.** Posts, likes, replies, follows and alerts go through the shared posts foundation; private messages are addressed by username. Only the appearance the player picks for themselves (avatar gradient, emoji, banner) stays on the phone, because the server only accepts an image address, not a colour.

**No stories.** BirdBy has none, by design.

</details>

<details>

<summary>TickTok</summary>

Short-video app: For You and Friends feeds, activity, profile, inbox.

**Real:** the video posts, likes, comments, follows and the activity tab, all through the shared posts foundation. Private messages are real too, addressed by username.

**Mockup:** the **Lives tab**. There is no live broadcast on the server side for TickTok, nothing to stream and no stream to receive. It exists so the app looks like the thing it is modelled on. If your players ask for live video, point them at InstaPic, which has a real one.

**Also local, on purpose:** saved videos and likes on individual comments. The server tracks neither, and they concern only the player.

</details>

<details>

<summary>PlayTube</summary>

YouTube-style app: home feed, shorts, subscriptions, library, channels, comments and an inbox.

**Real:** the account only. Sign-up, sign-in and reset-by-code work exactly like the other social apps, and the credentials can be saved to the byCloud keychain.

**Mockup: everything else.** Videos, channels, subscriber counts, comments and private messages are seeded content held in the phone's local storage. PlayTube is absent from the shared posts whitelist, from the private-messages whitelist and from the account search index. Nothing a player does in PlayTube reaches another player.

**One genuinely working piece:** pasting a YouTube link into the search field plays it in an embedded player, and three seeded entries carry a real video id. Playback happens in that player's own browser only; nobody else hears or sees it.

{% hint style="info" %}
This is the app most likely to generate a ticket. If your server advertises "a working YouTube", say plainly that PlayTube is a mockup with a link player. It is the only social app in the pack whose content does not travel.
{% endhint %}

</details>

## Media and capture

| App | `id` | Status | Stored in | Server file |
| --- | ---- | ------ | --------- | ----------- |
| Camera | `camera` | Networked | `nash_phone_gallery` | `server/apps/gallery.lua`, `server/services/upload.lua` |
| Photos | `gallery` | Personal (server) | `nash_phone_gallery` | `server/apps/gallery.lua` |
| Music | `music` | Networked (proximity audio) | none | `server/apps/music.lua` |

<details>

<summary>Camera</summary>

Photo, video and landscape modes, with a live viewfinder showing the real game view, mouse aiming, selfie mode, zoom and flash.

**Real.** The viewfinder is the game render, painted into a canvas through the CEF game-view plugin. What the player frames is what gets captured. When the game stream is not available, capture falls back to `screenshot-basic` (a full-screen 16:9 shot), which is a soft dependency checked at runtime.

{% hint style="danger" %}
**`config/upload.lua` is mandatory.** The phone stores a *link* to an image, never the image. Until an API key is filled in, the camera refuses to take pictures, warns the player, and prints a red line in the server console.

`config/upload.lua` is a **server script and must stay one**. Never move it to `shared_scripts`: a shared script is downloaded into every connecting player's cache and readable with any text editor, and your API key would become public.
{% endhint %}

Only `fivemanage` is supported as a provider. Discord webhooks are deliberately not supported: since December 2023 Discord signs its CDN URLs and they expire after 24 hours, so a photo album turns into a grid of broken thumbnails the next day.

**Configuration** in `Config.Camera` (`config/main.lua`):

| Key | Default | Meaning |
| --- | ------- | ------- |
| `Photo.LongEdgePx` | `1920` | long edge of the still, so 1440x1920 in photo mode |
| `Photo.Quality` | `0.8` | webp quality, 0.0 to 1.0 |
| `Recording.MaxDurationSeconds` | `30` | auto-stop |
| `Recording.Fps` | `30` | frames captured per second |
| `Recording.Bitrate` | `1200000` | bits per second, roughly 150 KB per second |
| `Recording.LongEdgePx` | `720` | long edge of the **video**, accepted range 240 to 1080. The heaviest setting for in-game performance: video is encoded live while GTA V runs |
| `Recording.Microphone` | `true` | record the filming player's voice. `false` removes the mic button entirely |

**Limits.** The game's own audio can never be recorded: the embedded browser never receives GTA's audio mix. Only the player's microphone can be captured. There is no AR mode.

**Uploads never cross the FiveM network.** The client asks the server for a single-use presigned upload address and posts the file straight to the host. Sending the image through a network event would blow past FiveM's 393,216-byte reliable event limit and kick the player on every photo.

</details>

<details>

<summary>Photos (`gallery`)</summary>

Camera roll: grid, full-screen viewer, video playback, favourites, deletion and a share sheet.

**Real, private per character.** Rows live in `nash_phone_gallery`. Every media carries a full-resolution `url` and a smaller `thumb`, generated by the camera at capture time as a second file. Serving full-resolution images in a 120 px grid is what makes an embedded browser that shares memory with the game fall over after thirty photos.

**The share sheet does real things.** A contact really receives the image in their conversation; a nearby player really receives it over AirDrop, inside `Config.NearShare.radius`.

**Limits.** Media taken before thumbnails existed, or through the `screenshot-basic` fallback path, have no `thumb` and fall back to the full-resolution file in the grid.

</details>

<details>

<summary>Music</summary>

Library, playlists, favourites, player, and playback of a pasted link.

**Real, and it is proximity audio.** The phone cannot play an audio file on its own, so playback goes through **`xsound`**, an optional dependency. The server triggers the sound for all clients and then moves it to follow the player: **people around you hear what you listen to**, like a real phone speaker. Without `xsound` running, the app says so instead of pretending to play.

**Configuration** in `config/music.lua`. The library ships **empty on purpose**: no music is included, neither for licensing reasons nor to impose taste. While it is empty the app shows a screen pointing back at this file.

- `tracks`: `id` (unique and stable, it is what player favourites reference, renumbering moves everybody's favourites), `title`, `artist`, `album`, `duration` in seconds, `url`, optional `grad` (two CSS colours for the cover).
- `playlists`: `name`, a list of track `id`s, optional `grad`. A playlist referencing only missing tracks is not displayed rather than shown empty.

A track without a URL is not displayed, and neither is one whose URL does not start with `http://` or `https://` or exceeds 512 characters. The goal is that a player can never tap a track that will not play: a missing track is almost always a mistyped address.

**Players keep pasted-link playback with no configuration at all**: the search bar accepts an address.

### Private listening (earbuds)

`Config.Earbuds` in `config/main.lua` turns speaker output into private listening: only the wearer hears anything.

| Key | Default | Meaning |
| --- | ------- | ------- |
| `enabled` | `true` | `false` removes the feature and hides the setting |
| `itemName` | `'phone_earbuds'` | the inventory item. **The resource creates no items**; create it in your inventory, `sql/item_earbuds.sql` covers the default ESX inventory |
| `requireItem` | `true` | must the item be carried for the earbuds to work. `false` means once paired, they work forever |

Using the item once pairs it; using it again connects and disconnects it, like a real pair. The item is never consumed. The setting also lives in Settings > Audio output. Without `xsound` there is no music to route, so the setting stays visible but does nothing.

</details>

## Utilities

| App | `id` | Status | Stored in |
| --- | ---- | ------ | --------- |
| Notes | `notes` | Personal (server) | `nash_phone_appdata` |
| Reminders | `reminders` | Personal (server) | `nash_phone_appdata` |
| Calendar | `calendar` | Personal (server) | `nash_phone_appdata` |
| Clock | `clock` | Personal (local) | phone local storage |
| Calculator | `calculator` | Personal | nothing to store |
| Compass | `compass` | Live game data | nothing to store |
| Maps | `maps` | Live game data | `config/maps.lua` |
| Weather | `weather` | **Mockup** | nothing |
| Health | `health` | **Mockup** | nothing |
| Stocks | `stocks` | **Mockup** | nothing |
| Browser | `safari` | **Mockup** | nothing |
| Snake | `snake` | Personal | nothing |
| Tic-Tac-Toe | `tictactoe` | Personal | nothing |

<details>

<summary>Notes, Reminders, Calendar</summary>

Three apps with no sharing at all, saved server-side per character as one JSON block per player and per app in `nash_phone_appdata` (`server/apps/appdata.lua`).

**Real and persistent.** They used to live only in the client cache, which belongs to the FiveM client and not to the character: a player clearing their cache or moving to another computer lost everything. They now follow the character.

**Limits worth knowing:**

- Each block is capped at **256 KB of JSON**. A note is a few hundred bytes, so a thousand notes fit; the cap exists because the payload comes from the NUI and a modified client could otherwise write a huge block on every keystroke.
- Only whitelisted app keys are accepted: `notes`, `calendar`, `reminders`, `contacts`. Anything else is refused.
- Writes are debounced by 700 ms and **never sent before a load has succeeded**. If the load fails, the app stays read-only until the next open. That is deliberate: better not to save than to overwrite a player's content with nothing.
- In game these three apps start **empty**. The demo notes, events and lists are only seeded in the web demo build.

</details>

<details>

<summary>Clock</summary>

World clock, alarms, stopwatch and timer.

**Functional, but local.** The stopwatch and timer work. Alarms work: each one carries its own ringtone, picked from the same list as Settings, rings for one minute in the Dynamic Island, and offers a 9-minute snooze.

**Two limits to state clearly.**

1. **Alarms are stored in the phone's local storage, not in the database.** They survive a NUI reload and a reconnect on the same machine, but not a cache wipe or a different computer. The key is cleared when the character changes, so a second character does not inherit them. There is no server-side alarm code at all.
2. Alarms are evaluated against the phone's displayed time. With the in-game clock setting on, that is **GTA time**, so an alarm set for 07:00 rings at 07:00 in game.

The world-clock cities are GTA universe names (Liberty City, Vice City and so on) with fixed offsets. They are a nod, not real time zones.

</details>

<details>

<summary>Calculator</summary>

Real and self-contained. Chained operations, percentages, sign and decimals, iOS-style repeat on `=`. No server, nothing to configure, nothing stored.

</details>

<details>

<summary>Compass</summary>

**Reads the live game.** Heading in degrees, cardinal direction, altitude and coordinates all come from the player's camera in game, pushed while the app is open and stopped when it closes.

The region shown next to the city ("Los Santos, San Andreas") comes from `Config.City.name` and `Config.City.region` in `config/settings.lua`. Leave `region` empty to show only the city.

</details>

<details>

<summary>Maps</summary>

Points of interest on the city map, with distance and travel time computed from the player's real position, plus a real GPS waypoint.

**Real.** Selecting a place sets an actual GTA waypoint. Everything the player sees is configured in `config/maps.lua`; nothing is hard-coded in the screens.

**World bounds** convert game coordinates into a 0..100% position on the tiles, and are also used by location sharing in Messages. Shipped as `minX = -5730.0`, `maxX = 6270.0`, `minY = -4000.0`, `maxY = 8000.0`.

{% hint style="warning" %}
The two spans must stay equal. Map tiles are square: one tile covers as many metres wide as tall. If `maxX - minX` differs from `maxY - minY`, the projection stretches one axis and places drift further the further they are from the centre. Here both spans are 12000. To fine-tune, shift **both** bounds of the same axis by the same amount.
{% endhint %}

**Travel estimates** (`Config.Map.Travel`): `roadFactor = 1.35` (1.0 would be straight-line), `speedKmh = 50.0`, `refreshSeconds = 5` while the app is open.

**Categories** (`Config.Map.Categories`) give each place its colour and default icon: shipped with `place`, `hospital`, `police`, `garage`, `bank`, `airport`. `chip = true` adds a filter shortcut under the search bar. Leave `label` empty for the six original categories, which the phone already translates.

**Places** are given in **game coordinates**. Stand where you want the marker and type `/phonepos` in chat; the line to paste is printed in F8.

</details>

<details>

<summary>Weather</summary>

**Mockup.** Fixed forecast: one current condition, an hourly strip and a multi-day outlook, all written as constants in the app. There is no weather sync in the Lua at all, so the app never reflects the actual in-game weather. The only thing that follows your configuration is the city name (`Config.City.name`).

</details>

<details>

<summary>Health</summary>

**Mockup.** Activity rings, steps, heart rate, sleep and distance are constants in the app. Nothing reads the player's actual health, stamina or movement, and there is no Health-related Lua on either side.

</details>

<details>

<summary>Stocks</summary>

**Mockup.** Prices are generated locally by a deterministic pseudo-random walk, so they are stable across renders but have no relationship to any market, in game or out. The company names are GTA universe nods. There is no server code and no link to any economy or banking resource.

</details>

<details>

<summary>Browser (`safari`)</summary>

**Mockup, and deliberately inert.** The start page and chrome look like a browser, but the whole screen carries `pointer-events: none`: nothing is clickable, nothing navigates, and no page can ever be loaded. This is a product decision, not an unfinished feature; do not plan a server around it.

The one configurable part is the first bookmark and the first "frequently visited" row, which come from `Config.Brand.site` in `config/main.lua`: `label` (short name under the icon), `url` (displayed address, no navigation happens) and `title`, a value per language keyed by language code.

The app id, component and CSS classes still contain the word `safari`. They are identifiers; renaming them would break the home screen, the dock and the catalogue.

</details>

<details>

<summary>Snake and Tic-Tac-Toe</summary>

Two real, playable games with no server side. Snake speeds up as the score climbs. Tic-Tac-Toe plays against an AI that blocks and punishes mistakes, or two players on the same phone. Nothing is stored, nothing is shared, no leaderboards.

Both are App Store downloads by default and are filed under the Games category, credited to `<Config.Brand.studio> Arcade`.

</details>

## System

| App | `id` | Status | Stored in | Server file |
| --- | ---- | ------ | --------- | ----------- |
| Settings | `settings` | Personal (server) | `nash_phone_settings` | `server/apps/settings.lua` |
| App Store | `appstore` | Personal (server) | `nash_phone_home` | `server/apps/home.lua` |
| Wallet | `wallet` | Networked | `nash_phone_bank` + framework accounts | `server/apps/bank.lua` |
| Passwords | `passwords` | Personal (local + cloud backup) | phone local storage, `nash_phone_icloud_backup` | `server/apps/icloud.lua` |

<details>

<summary>Settings</summary>

The phone's own settings: profile, wallpaper and lock screen customisation, ringtones, language, dark mode, Face ID and passcode, audio output, storage, byCloud account, streamer mode.

**Real.** Everything is persisted per character in `nash_phone_settings` as a JSON block, cached server-side with idle eviction.

**Language.** `Config.Locale` in `config/main.lua` (shipped `'fr'`) sets both the language of out-of-phone notifications and the starting language of a phone nobody has touched. The player overrides it in Settings > General > Language and keeps their choice. `Config.ExtraLocales` (shipped empty) lists the extra languages offered in that menu, on top of French and English. Any code with a matching `locales/<code>.json` file is accepted; nothing to compile.

**Brand.** `Config.Brand.name` (shipped `'NASH'`) replaces the word NASH everywhere a player sees it: the network operator in Settings > SIM, the account identifier at the top of Settings, the device name in Settings > General > About, the Settings footer, and the sender of social verification SMS.

**Storage page.** Per-app usage is computed from a real model: photos and videos priced by duration, apps by install size.

**First-run setup** is a separate flow (`config/setup.lua`, `server/apps/setup.lua`), shown once per character the first time they open the phone: greeting, language, country, appearance, byCloud account, identity, privacy, Face ID, passcode. The identity screen only appears when a byCloud account is being created.

Nothing is applied until the end, and the values it collects (language, appearance, Face ID, passcode) are written to `nash_phone_settings`, the same place the lock screen and Settings already read them from; the identity belongs to the byCloud account, not to the setup table. `nash_phone_setup` only remembers **where the player got to**, so somebody who disconnects halfway resumes instead of starting over. Existing characters are marked as already set up and never see it.

Shipped as `Config.Setup.enabled = true` and `skippable = false`. `exports.nash_phone:ResetPhoneSetup(source)` replays it; the `/phonesetupreset` command only exists when `Config.Debug` is on.

</details>

<details>

<summary>App Store</summary>

Browsing, install and uninstall for the apps marked `store = true`, plus a Today tab and an account page listing what the player installed.

**Real, and free.** Installs are persisted in `nash_phone_home` alongside the home layout and dock, so a player finds their apps again after a reconnect. **There is no in-game money price for stock apps**: the App Store never renders a price. Custom apps declare a `price` field for LB Phone compatibility, but nothing in the phone charges for it.

**Generated store pages.** When you push a preinstalled system app into the store (`store = true` on an app that ships preinstalled), it has no hand-written store page. The phone composes one from what it knows: the icon, the translated or overridden name, the studio from `Config.Brand.studio`, and the icon gradient in place of screenshots.

Store listings credit `<Config.Brand.studio> Arcade` for games, `<studio> Labs` and `<studio> Finance` for the utility apps, so setting `studio = 'Vespucci'` gives "Vespucci Arcade".

</details>

<details>

<summary>Wallet</summary>

Balance, send money, recent activity and a searchable, paged transaction history.

**Real.** Balances are read live from the framework (`bank` and `cash` accounts on ESX, `money.bank` / `money.cash` on QBCore and Qbox). Transfers really debit and credit through the framework; the phone's own `nash_phone_bank` table holds the history, two rows per transfer, one per side.

**Configuration.** None of its own. It follows `Config.Framework` in `config/bridge.lua` (`'auto'` by default, detection order Qbox, QBCore, ESX).

**Limits.**

- A transfer requires the **recipient to be online**. An offline number returns `number_offline` and nothing moves.
- The memo is capped at 128 characters, matching the column.
- History pages are 25 rows by default, 50 maximum, and the cursor is `(created_at, id)` because a transfer writes two rows with the same timestamp.
- This is an account-and-transfer app, not a card wallet. There are no card objects, limits or PINs in it.

{% hint style="danger" %}
Changing `Config.Framework` on a live server orphans data. Every table is indexed on the identifier the framework hands out, and that identifier has a different shape from one framework to another (ESX licence, QB citizenid). Players would find an empty phone and their numbers, contacts, messages and photos would become unreachable.
{% endhint %}

</details>

<details>

<summary>Passwords</summary>

A Face ID-locked keychain: entries with site, username, password, favourites, search and an alphabetical index.

**Real.** Entries are saved in the phone's local storage, and backed up to the **byCloud** account when the player has one, in `nash_phone_icloud_backup`. Signing into a byCloud account restores the keychain, which is what makes it portable to another phone.

**How entries get there.** After creating an account in a social app, the phone offers to save the credentials. The player can also add entries by hand.

**Not in the App Store.** It is a stock app, preinstalled and absent from the store catalogue.

**The unlock is theatre, the storage is not.** The app does not draw its own face recognition; it drives the Dynamic Island's Face ID animation, with a local fallback of the same duration if the island is not available. It is a lock in the roleplay sense, not a cryptographic one.

**Consequence worth stating to players:** because a byCloud account carries both the mailbox and the keychain, someone who steals byCloud credentials gets both. See [Mail](#communication).

</details>

## Custom apps

You can add your own apps in two ways, both documented in `config/custom_apps.lua`.

1. **In the config file**, in `Config.CustomApps`. Simplest, and the app appears on every phone on the server.
2. **From your own resource**, with the client export, once the player is loaded:

```lua
exports['nash_phone']:AddCustomApp({
    identifier = 'ma_banque',
    name = 'Ma Banque',
    ui = GetCurrentResourceName() .. '/ui/index.html',
})
```

Use the export form when your app has a UI, because its HTML has to live in your resource.

| Field | Meaning |
| ----- | ------- |
| `identifier` | **Required.** Unique key, never shown to the player |
| `name` | Display name under the icon |
| `description`, `developer`, `images`, `size` | Shown on the App Store page. `size` is in KB and purely cosmetic |
| `icon` | Icon URL. For an image in your resource: `'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png'` |
| `ui` | Path to your HTML, **prefixed with your resource name**: `'my-resource/ui/index.html'`. Without it the app only fires `onUse` when tapped |
| `defaultApp` | `true` (default) = on the home screen at first start. `false` = App Store download |
| `game` | `true` files it under the App Store's Games category |
| `landscape` | `true` displays the app in landscape |
| `keepOpen` | `true` keeps the phone open when the app is opened. For apps that take over the screen, a camera for instance |
| `onUse` | Client-side function called when the app opens |
| `onServerUse` | Server-side function called when the app opens, receives the player's `source` |

**LB Phone compatibility.** The field names above are LB Phone's, deliberately: an app written for LB Phone declares itself here without changing its table. Two things to know before relying on it:

- The app's Lua calls `exports['lb-phone']`. If you can edit the app, replace that string with `exports['nash_phone']` and you are done. If the app is escrowed and cannot be modified, you need the `nash_lbcompat` bridge, shipped separately.
- An app is only as portable as the exports it uses. Only some of the LB Phone export surface is actually implemented; the rest return neutral values, so the app launches without crashing but without data.

{% hint style="warning" %}
`price` is accepted in the table and forwarded to the interface for LB Phone compatibility, but **nothing charges for it**. A custom app with a price is still free to install.
{% endhint %}

Custom apps are also reachable from `exports.nash_phone:SendCustomAppMessage(src, identifier, data)` on the server side, and the phone hosts their UI in an isolated frame (`custom_app_bridge/nash-bridge.js`).

## Decorative apps, on purpose

The honest list, with the real part of each app called out next to it. No configuration turns the decorative column into anything else.

| App | `id` | What is decorative | What is real |
| --- | ---- | ------------------ | ------------ |
| Browser | `safari` | Everything. The screen is inert, nothing is clickable, no page loads | The bookmark and "frequently visited" row read `Config.Brand.site` |
| Weather | `weather` | The whole forecast, written as constants. No link to in-game weather | The city name, from `Config.City.name` |
| Health | `health` | Rings, steps, heart rate, sleep, distance, all constants | Nothing |
| Stocks | `stocks` | Prices generated locally by a seeded random walk | Nothing |
| PlayTube | `playtube` | Videos, channels, comments, subscriptions, private messages. All local, none of it travels | The account (sign-up, sign-in, reset by code) and the pasted-YouTube-link player |
| TickTok, Lives tab | `ticktok` | The Lives tab only | The rest of the app: feed, likes, comments, follows, activity, private messages |

Two more things are worth telling your players even though the apps themselves are real. The Clock's world-clock cities are GTA universe names with fixed offsets, not real time zones, and its **alarms live in the phone's local storage, not in the database**. Snapz, InstaPic and BirdBy let the player pick an avatar gradient and emoji that never leave their own phone, because the server only stores an image address; Snapz otherwise renders the character's real RP head.

## What to configure, app by app

A checklist of everything that has to be filled in before an app is useful. Anything not listed here works out of the box.

| App | File | What to do |
| --- | ---- | ---------- |
| Camera, Photos, all social apps | `config/upload.lua` | **Mandatory.** A Fivemanage API key. Without it the camera refuses to shoot and no photo can be posted anywhere |
| Music | `config/music.lua` | Fill `tracks` and `playlists`. The library ships empty. Requires `xsound` running |
| Music, private listening | `config/main.lua` | `Config.Earbuds.itemName`, and create the item in **your** inventory. `sql/item_earbuds.sql` for default ESX |
| Maps | `config/maps.lua` | World bounds, categories, and the places themselves in game coordinates. Use `/phonepos` in chat to read a position |
| Services | `config/services.lua` | One entry per company, with the exact framework `job` name, and a PNG in `service_icons/` |
| Mail | `config/main.lua` | `Config.Mail.domain`, and decide on `Config.Cloud.crossLogin` |
| VideoCall, InstaPic Lives | `config/main.lua` | `Config.VideoCall.iceServers`. Add a TURN server if your players sit behind CGNAT |
| InstaPic Lives | `config/social.lua` | `Config.Social.Live.maxViewers` and `defaultAudience` |
| Snapz, InstaPic stories | `config/social.lua` | `Config.Social.StoriesAudience` per app |
| Snapz map | `config/social.lua` | `Config.Social.SnapMap`, in particular `blurMeters` and `requireMutual` |
| Social notification cleanup | `config/social.lua` | `Config.Social.Notifs`: `purgeOnJoin = 'read'`, `keepReadDays = 2`, `keepDays = 7` |
| Phone, all calls | `config/main.lua` | `Config.Voice.provider` and `Config.Voice.Speaker`. Requires a voice resource, `pma-voice` by default |
| Weather, Compass, App Store, InstaPic | `config/settings.lua` | `Config.City.name` and `Config.City.region` |
| Every app | `config/apps.lua` | `enabled`, `name`, `store` per app |
| Custom apps | `config/custom_apps.lua` | Or the `AddCustomApp` export from your own resource |

{% hint style="info" %}
Set `Config.City.name` **before you open your server**. Live screens follow a change immediately, but an InstaPic post records the city name in its `place` field at the moment it is published: renaming the city later does not rewrite posts that already exist.

The other GTA universe names stay put whatever you write there: the Clock's time zones (Liberty City, Vice City), the Stocks companies (Maze Bank, Lifeinvader) and the street names on the Maps tiles. They are nods, not city names, and changing them would mean rewriting the content of those apps.
{% endhint %}
