# config/social.lua

What players see of each other in the social applications: story reach, notification cleanup,
the Snapz map, and InstaPic live streams.

File: `config/social.lua`, loaded as a shared script after `config/main.lua`.

Nothing in this file is a technical setting. Everything below is visible on screen, in the
game. Which social apps exist at all, and whether they start installed or have to be
downloaded from the App Store, is set in [config/apps.lua](config-apps.md).

All four blocks are read live, never cached: a `restart nash_phone` applies a change without
anyone having to reconnect.

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
to the people watching. The image never passes through the server: it goes straight from one
player to the other, exactly like a video call. The server only relays the handful of messages
the two browsers need to find each other.

```lua
Live = {
    enabled = true,
    maxViewers = 8,
    defaultAudience = 'followers',
},
```

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `enabled` | `true` | `false` keeps the button visible but the server refuses to start: the app shows a clear refusal instead of a dead button |
| `maxViewers` | `8` | Most viewers per live. Clamped to the 1-24 range |
| `defaultAudience` | `'followers'` | Value proposed on the launch screen. `'everyone'`, `'followers'` or `'none'` |

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

`defaultAudience` is only the value proposed on the launch screen; the player changes it at the
moment they go live.

| Value | Who is told |
| ----- | ----------- |
| `'everyone'` | Everyone who has an InstaPic account and is connected |
| `'followers'` | Their followers only |
| `'none'` | Nobody |

The notification stays **inside the phone**. It never draws over the game screen, unlike an
incoming call.

### Fixed limits

These are not configurable:

| Limit | Value |
| ----- | ----- |
| Guests on screen | The host plus 3. The panel is tall and narrow, past that nothing is visible |
| Live title | 60 characters |
| Comment | 200 characters |
| Chat lines kept for latecomers | 60 |

A live has **no SQL table**, by design: it only means anything while the broadcaster is
connected. An interrupted live is a finished live, and nothing is restored after a restart.

The list of ongoing lives only ever returns lives from accounts the requester already follows,
plus their own. Returning "every live on the server" would hand out a directory of connected
players, which the whole social layer refuses to do.

## Checking what the server applied

With `Config.Debug = true` in [config/main.lua](config-main.md):

| Command | What it shows |
| ------- | ------------- |
| `/phoneconfig` | Everything this file produced, including the raw value next to the applied one, so a typo is visible |
| `/phonesnapmap [handle]` | Why the Snapz map shows, or does not show, someone: lock by lock |
| `/phonesocialnotifs <app> [days]` | Writes notifications that are born already old, so the cleanup can be observed without waiting a week. One read row and one unread row per pass, which is what makes `purgeOnJoin` visible |
| `/phonesocialnotifs purge` | Runs the age sweep immediately |
| `/phonelive` | Ongoing lives: host, viewer count, guests, title |
| `/phonelivewho` | Who would be notified if you went live right now |
| `/phonesocial` | Your own social accounts and their handles |
