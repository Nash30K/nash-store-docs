# config/social.lua

What players see of each other in the social applications: story reach, notification cleanup,
the Snapz map, InstaPic live streams, and the Cindr dating app.

File: `config/social.lua`, loaded as a shared script after `config/main.lua`.

Nothing in this file is a technical setting. Everything below is visible on screen, in the
game. Which social apps exist at all, and whether they start installed or have to be
downloaded from the App Store, is set in [config/apps.lua](config-apps.md).

All five blocks are read on each request, never cached by the phone: a `restart nash_phone`
applies a change without anyone having to reconnect.

## Config.Social.StoriesAudience

Stories are the circles at the top of a feed: an image published for 24 hours that disappears
on its own. This setting decides **which ones a player sees**, application by application.

```lua
StoriesAudience = {
    instapic = 'everyone',
    snapz    = 'following',
},
```

| Value | Effect |
| ----- | ------ |
| `'everyone'` | Every story published on the server in the last 24 hours appears, including from accounts the player does not follow |
| `'following'` | Only stories from accounts the player follows, plus their own. A brand new account opens an empty carousel |

`'everyone'` is the small-town behaviour: a player who has just created an account has
something to look at immediately, and a published story gets seen. `'following'` suits a
server that wants links between characters to be made in game before they exist in the phone.

### What this setting does not change

- The **feed** stays limited to what the player follows, in every application. Only stories are
  affected here.
- A **private account** (a per-player toggle in their own profile) only shows its stories to
  its followers, even under `'everyone'`. The setting opens up only what players agreed to
  make public.
- There is still **no account directory**. You only see people who published something in the
  last 24 hours, and only because they published it. Writing to someone still requires knowing
  their handle and searching for it.
- An account cannot have more than **40 live stories** at a time. Past that, publishing is
  refused until the oldest ones reach 24 hours. Without this cap one player could fill the
  whole server's carousel. This limit is not configurable.

ChatApp is absent from the list, and does not belong in it: its feed is derived from the
address book, exactly like its conversation list. A contact added in the Contacts app makes
their posts appear on their own, a stranger never appears at all.

{% hint style="info" %}
A misspelled value is not applied. The phone keeps the shipped value and writes one line to
the server console, for example:

```
[nash_phone] config/social.lua : StoriesAudience.instapic = "everybody" inconnu, valeur ignorée (everyone appliqué). Valeurs acceptées : 'everyone' ou 'following'.
```

Writing `StoriesAudience = 'everyone'` instead of a per-app table applies to nobody, and is
reported the same way.
{% endhint %}

## Config.Social.Notifs

The list behind the heart or bell tab of a social app, one line per reaction received: "X
liked your post", "X commented", "X started following you". These three settings apply to the
**four applications at once** (InstaPic, Snapz, BirdBy, TickTok). There is not one per app.

This has nothing to do with the phone's banners (incoming SMS, missed call, transfer), which
are configured in [config/notifications.lua](config-notifications.md).

```lua
Notifs = {
    purgeOnJoin = 'read',
    keepReadDays = 2,
    keepDays = 7,
},
```

### Why these settings do not look like the banner ones

A phone banner is only ever written for an **online** player. Nothing accumulates while they
are away, so wiping everything when they connect costs them nothing: that is why
`config/notifications.lua` gets away with a yes or no.

A social notification is written **even if its recipient is disconnected**. Someone liking a
post at three in the morning leaves a row that waits for the player to come back. Wiping
everything on connect would therefore delete rows nobody ever read.

### purgeOnJoin

| Value | Effect on connect |
| ----- | ----------------- |
| `'read'` | Deletes the player's **already read** notifications, across all four apps at once. Anything that arrived while they were away and has not been opened survives |
| `'all'` | Deletes everything, seen or not. Tabs are empty at the start of every session |
| `false` | Deletes nothing on connect. Only the two retention values below still clean up |

Shipped on `'read'`: the only value that empties the list without ever making something
disappear that the player has not seen.

"Read" here means the player **opened that tab** since. The phone marks the whole list read
the moment it is opened, it does not track row by row. Glancing at it once is therefore
accepting that everything it contained goes at the next connection.

There is no purge on disconnect, and that is not an oversight: at the moment a player leaves,
nothing says what will reach them overnight. The cleanup happens when they come back, once
only seen rows are left.

{% hint style="warning" %}
An unknown value is **not** treated as `'all'`. The phone falls back to `'read'` and writes one
console line. The likeliest mistake is `purgeOnJoin = true`, written by imitation of
`config/notifications.lua` where that setting is a plain boolean.
{% endhint %}

### keepReadDays and keepDays

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `keepReadDays` | `2` | A **read** notification older than this many days is deleted. `0` = unlimited |
| `keepDays` | `7` | Any notification, read or not, older than this many days is deleted. `0` = unlimited |

`keepReadDays` catches what the connect purge cannot see: players who never reconnect, and
read rows that pile up when `purgeOnJoin` is `false`.

These two durations sweep **the whole table**, all players included, even those who will never
come back. `purgeOnJoin` only touches **one** phone, at the moment its owner arrives. They are
not two ways of doing the same thing, they complement each other.

The sweep runs once an hour, starting one minute after the resource boots. If both durations
are `0` it stops for good, since there is nothing left to sweep.

{% hint style="info" %}
The tab heads its list with a fixed title, "This week". That is a label, not a grouping by
date. At 7 days it tells the truth. Raise `keepDays` and rows older than a week will appear
under that title.
{% endhint %}

{% hint style="warning" %}
Turning everything off (`purgeOnJoin = false` and both durations at `0`) is allowed, but
nothing is left to empty those lists. They grow for as long as the player plays and the table
grows without end. The only remaining guard is not a setting: the tab only ever displays the
**50 most recent** notifications. The screen stays readable, the database does not.
{% endhint %}

### What is never deleted

A notification is an **alert**, not the content. The post stays published, the comment stays
under the post, the like stays counted in its total, and a follow removed from the
notification list is still a follow: that person still follows the account and still sees its
posts. Only the announcement disappears.

## Config.Social.SnapMap

The map tab of Snapz. It does not show scenery: it shows where the player's friends **actually
are in the game world**, among those who agreed to be seen. It is the most sensitive setting in
this file, because an in-game position is in-game information.

Only Snapz uses this map. No other social app has one.

```lua
SnapMap = {
    enabled = true,
    requireMutual = true,
    refreshSeconds = 5,
    blurMeters = 0,
    maxFriends = 50,
},
```

### The four locks

For a dot to appear on someone's map, all four conditions must be true at once:

1. **Both follow each other.** A one-way follow is not enough, so nobody can start following
   someone in order to track them.
2. **The person said yes.** The map asks the question the first time it is opened and shows
   nothing until it is answered. Hiding again takes one tap, from the map, at any time.
3. **They are connected.** A position is never stored in the database: it is read from the game
   at the moment the map refreshes and disappears with the disconnection. Nobody can find out
   where someone stopped playing.
4. **You left the map enabled** below.

None of the four can be bypassed by the player. The query that builds the map starts from the
requester's own subscriptions, so it cannot enumerate accounts: it can only show people the
player already follows and who follow back, which means people they met in game.

### Settings

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `enabled` | `true` | `false` keeps the tab but shows nobody, and the phone says so plainly instead of leaving the player in front of an empty map |
| `requireMutual` | `true` | Lock 1 above. `false` means following someone is enough to see them, without their knowing |
| `refreshSeconds` | `5` | How often the map asks for positions, **only while the player is looking at it**. Values below 3 are clamped to 3 |
| `blurMeters` | `0` | `0` = exact position. Above zero, positions are snapped to a grid of that size |
| `maxFriends` | `50` | Most friends shown at once. Clamped to at least 1 |

Set `enabled = false` on a server that considers knowing another character's position to be
metagaming, even between friends and even with their consent.

With `requireMutual = false`, only the target's own sharing choice still protects them.

`blurMeters` is the compromise for a server that wants to keep the map as a social link without
making it a tracking tool: the friend appears in the right neighbourhood, never at the right
door. 150 metres is roughly one city block. **The blur is applied by the server**, so the exact
position never leaves the machine and a modified client cannot get anything more precise out
of it.

`maxFriends` is a load bound, not a privacy one. Past that number the map becomes unreadable
and every opening would make the server work for overlapping dots. A player with more
connected friends than this sees the first ones, never an empty map.

{% hint style="info" %}
`SnapMap` is filled setting by setting. Written as a single value (`SnapMap = false`, by
imitation of `enabled`) it cannot be read, the shipped values take over, and the map would show
positions on a server that had just closed it. The phone reports that case once in the console.
{% endhint %}

## Config.Social.Live

InstaPic live streams. A live broadcasts the **actual game view** of the player who starts it
to the people watching, with the microphone of the host and of the guests on screen. Picture
and sound never pass through the server: they go straight from one player to the other, exactly
like a video call. The server only relays the handful of messages the two browsers need to find
each other.

```lua
Live = {
    enabled = true,
    maxViewers = 8,
    audience = 'followers',
    microphone = true,
},
```

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `enabled` | `true` | `false` keeps the button visible but the server refuses to start: the app shows a clear refusal instead of a dead button |
| `maxViewers` | `8` | Most viewers per live. Clamped to the 1 to 24 range |
| `audience` | `'followers'` | Who is **notified** when a live starts: `'everyone'`, `'followers'` or `'none'`. An unknown value gives `'followers'` |
| `microphone` | `true` | Viewers hear the host and the guests on screen through their microphone, captured by the phone. `false` gives silent lives: no audio channel and no mic button. Any value other than `false` counts as `true` |

Lives carry picture and sound only when `Config.VideoCall.enabled` is `true` in
[config/main.lua](config-main.md#video-calls-facetime), which also sets the framing, resolution and
frame rate of the stream. With video calls off, lives run with comments and viewers only, and the
server console says so at startup.

### Sound

- **Only the microphone is sent.** The game's own sound never is: the embedded browser never
  receives GTA's audio mix. The sound travels with the picture, not through `pma-voice`, so a
  player with no route to the host (see [config/turn.lua](config-turn.md)) gets neither.
  `Config.Camera.Recording.Microphone` has no effect on lives.
- **Only the host and the guests on screen are heard.** A viewer's microphone is never captured.
- **Mic button.** The host and the guests get a button next to the camera flip button. It mutes
  and unmutes without reconnecting anyone.
- **No microphone.** When the microphone is refused, missing or has not answered after 5
  seconds, the live goes on with picture only and viewers see no error. The button then shows a
  red crossed-out mic: clicking it asks for the microphone again. A microphone that answers late
  is added to the live, and viewers lose the picture for a moment while the connection is
  rebuilt.
- **During a phone call.** The host and the guests stop sending picture and sound until they
  hang up. A viewer in a call stops hearing the live until they hang up.
- **Volume** follows the phone's volume (Control Center slider and side buttons). If the sound
  does not start on its own, it starts at the next click inside the live screen.

### Why the viewer cap is low

{% hint style="danger" %}
**This is not a comfort setting, it is a physical limit.** With no media server, the
broadcaster sends a **separate stream to each viewer**: their upstream bandwidth and their CPU
rise in proportion to the number of viewers. It is the same reason a call is capped at four
participants.

At 8, a player on fibre holds up without trouble. Past ten or so the broadcast degrades **for
everyone**, including viewers who were already connected, and the broadcaster starts dropping
frames in their own game. Raising this number does not make room for more people, it splits the
same capacity across more of them.
{% endhint %}

When the cap is reached the live stays open. Newcomers get a readable refusal, they do not cut
anyone off.

### Audience

`audience` is a server-wide setting: it applies to every live, and the player does not choose it.
It only decides who receives the "live started" notification. **Whatever the value, only the
broadcaster's followers can watch**: a player who does not follow the host is refused when
joining.

| Value | Who is told |
| ----- | ----------- |
| `'everyone'` | Everyone who has an InstaPic account and is connected |
| `'followers'` | Their followers only |
| `'none'` | Nobody |

The notification stays **inside the phone**. It never draws over the game screen, unlike an
incoming call.

An older file may still write `defaultAudience`: it is read the same way when `audience` is
absent. When both are filled with different values, `audience` wins and the console says so.

### Fixed limits

These are not configurable:

| Limit | Value |
| ----- | ----- |
| Guests on screen | The host plus 3. The panel is tall and narrow, past that nothing is visible |
| Live title | 60 characters |
| Comment | 200 characters |
| Comments per player | 5 every 5 seconds |
| Chat lines kept for latecomers | 60 |
| Starting a live | Once per minute per character, never during a call |

A live has **no SQL table**, by design: it only means anything while the broadcaster is
connected. An interrupted live is a finished live, and nothing is restored after a restart.

The list of ongoing lives only ever returns lives from accounts the requester already follows,
plus their own. Returning "every live on the server" would hand out a directory of connected
players, which the whole social layer refuses to do.

## Config.Social.Dating

Cindr, the dating app: a deck of profiles to like or pass, matches, and a conversation only
once both players liked each other. Whether the app exists at all, and whether it has to be
downloaded, is set in [config/apps.lua](config-apps.md) (`cindr`).

```lua
Dating = {
    enabled = true,
    deckSize = 12,
    minAge = 18,
    maxAge = 99,
    maxPhotos = 6,
    shareZone = true,
    undoSeconds = 300,
    reports = true,
},
```

| Setting | Default | Accepted range | Meaning |
| ------- | ------- | -------------- | ------- |
| `enabled` | `true` | - | `false` makes the server refuse every Cindr action. Profiles, matches and conversations are kept and come back when you turn it on again |
| `deckSize` | `12` | 1 to 50 | Profiles sent to the player at once |
| `minAge` | `18` | 18 to 99 | Youngest age a player can declare on their profile. It can never go below 18 |
| `maxAge` | `99` | `minAge` to 120 | Oldest age a player can declare |
| `maxPhotos` | `6` | 1 to 6 | Photos per profile |
| `shareZone` | `true` | - | `true` lets players show their neighbourhood on their profile, if they choose to. `false` hides it on every profile and stops saving it |
| `undoSeconds` | `300` | 0 to 3600 | How long the player can undo their last like or pass, in seconds. `0` removes the time limit; it does not disable undo |
| `reports` | `true` | - | `false` refuses reports. When on, each report is stored, printed in the server console and sent to the `cindr_report` log of [config/logs.lua](config-logs.md). There is no automatic sanction |

Values outside the accepted range are brought back to the nearest bound. A `Dating` block that is
not a table is ignored with one console line, and the shipped values apply.

Fixed limits, not configurable: 5 interests and a 500-character bio per profile, 30 deck
requests and 90 likes or passes per minute, 5 reports every 5 minutes per player. An undo is
refused once the two players have matched.

## Checking what the server applied

There is no command that prints this file back. What the server could not apply is reported in
the server console, once, with the value it used instead:

| Block | When the line is printed |
| ----- | ------------------------ |
| `StoriesAudience`, `Notifs`, `SnapMap`, `Dating` | The first time the setting is read after a restart: a block that is not a table, or an unknown value of `StoriesAudience` or `Notifs.purgeOnJoin` |
| `Live` | About two seconds after startup, when `audience` and `defaultAudience` disagree, or when lives are on while `Config.VideoCall.enabled` is `false` |

Everything else is visible in game: open the app and check what it shows. Cindr reports are
printed in the server console as they arrive, and stored in the `nash_phone_dating_reports`
table, which you can read with SQL.
