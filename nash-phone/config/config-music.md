# config/music.lua

The music library offered inside the Music app.

File: `config/music.lua`, loaded as a shared script.

## Read this before filling the file

The phone does **not play the audio inside its own interface**. Sound goes through `xsound`, as
**proximity audio**: the players around you hear what you are listening to, like a real phone
speaker (or only the player, with earbuds).

Three direct consequences:

- `xsound` must be running on the server. Without it the phone says so and does not pretend to
  play anything. The rest of the phone is unaffected.
- Every track needs an **address that xsound can read**: a direct link to an audio file, or a
  YouTube link depending on your version of xsound.
- A track **without an address is not displayed**, and neither is a track without a numeric
  `id`, or whose address does not start with `http://` or `https://` followed by a host name, or
  is longer than 512 characters. The goal is that a
  player can never tap a track that will not play. A track missing from the app is almost
  always a track whose address was mistyped.

The library ships **empty**, and that is deliberate: no music comes with the resource, neither
for licensing reasons nor to impose a taste. While it is empty, the app shows a screen pointing
back here.

Your players keep **paste-a-link playback**, which works with nothing configured at all: the
app's search bar accepts an address.

## Config.Music.tracks

```lua
Config.Music = {
    tracks = {
        { id = 1, title = 'Sleepless Night', artist = 'Kessler', album = 'Neon',
          duration = 213, url = 'https://example.com/music/sleepless-night.mp3',
          grad = { '#ff375f', '#ff8a5c' } },
    },
    playlists = {},
}
```

| Field | Required | Description |
| ----- | -------- | ----------- |
| `id` | **Yes** | A number, unique and **stable**. This is what gets stored in players' favourites: renumbering it moves everyone's favourites |
| `url` | **Yes** | `http://` or `https://`, 512 characters maximum |
| `title` | No | Trimmed to 80 characters |
| `artist` | No | Trimmed to 80 characters |
| `album` | No | Trimmed to 80 characters |
| `duration` | No | In **seconds**, used by the progress bar. A wrong value does not stop playback, it only offsets the display |
| `grad` | No | Cover art: two CSS colours. Omit it and a colour is derived from the title |

A duplicate `id` is ignored: the first entry wins, the second is dropped, so that nobody's
favourites move.

## Config.Music.playlists

Home-screen playlists. `tracks` holds `id` values from the list above.

```lua
playlists = {
    { name = 'Night', tracks = { 1, 3, 5 }, grad = { '#5e5ce6', '#ad6bff' } },
},
```

| Field | Required | Description |
| ----- | -------- | ----------- |
| `name` | **Yes** | Trimmed to 60 characters. An empty name means the playlist is dropped |
| `tracks` | **Yes** | List of track `id` values. Identifiers that do not resolve are silently skipped |
| `grad` | No | Cover art: two CSS colours |

A playlist that only references missing tracks is **not displayed**, rather than appearing
empty.

## Why a track does not show up

The library is cleaned once and cached at startup, so **restart the resource** after editing
this file.

A dropped track leaves no line in the console. Count the tracks in the Music app against the
entries of the file, then check each missing one against the rules above:

- `id` is a number, and no earlier entry uses the same one.
- `url` is filled in, starts with `http://` or `https://` followed by a host name, and is 512
  characters at most.

A playlist missing from the app has an empty `name`, or none of its `tracks` match a track that
survived.

## Playback settings

Proximity range, default and maximum volume, and whether music stops when the player dies are
declared as `Config.Music` in [config/main.lua](config-main.md). `config/music.lua` adds `tracks`
and `playlists` to that same table, so both files are read.

## Private listening

Wireless earbuds switch a player from the speaker to private listening, so only they hear the
music. That is an inventory item configured through `Config.Earbuds` in
[config/main.lua](config-main.md), not here. Without `xsound` there is no music to route
anywhere, so the setting stays visible but has nothing to do.
