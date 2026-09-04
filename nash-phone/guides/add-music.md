# Fill the music library

Put tracks and playlists in the Music app, so players have something to browse instead of an
empty library that points back at the configuration.

## Prerequisites

- The **`xsound`** resource started on your server
- A direct address for each track that xsound can read
- Everything happens in `config/music.lua`

{% hint style="info" %}
**Why an external address is required.** The phone cannot play an audio file by itself: its
interface runs in the FiveM CEF browser, which has no network access. Sound goes through
`xsound`, as **proximity audio:** players around you hear what you are listening to, like a
real phone speaker.

Without `xsound` started, the phone says so and does not pretend to play anything. The rest of
the phone works normally.
{% endhint %}

The library ships **empty**, deliberately: no music is bundled with the resource, neither for
licensing reasons nor to impose our taste on your server. While it is empty, the app shows a
screen that points back here.

Your players keep pasted-link playback either way: it works with no configuration at all,
because the app's search bar accepts an address.

## 1. Add tracks

```lua
-- config/music.lua
Config.Music = {
    tracks = {
        { id = 1, title = 'Nuit blanche', artist = 'Kessler', album = 'Néon',
          duration = 213, url = 'https://example.com/music/nuit-blanche.mp3',
          grad = { '#ff375f', '#ff8a5c' } },
        { id = 2, title = 'Downtown', artist = 'Kessler', album = 'Néon',
          duration = 187, url = 'https://example.com/music/downtown.mp3' },
    },
    playlists = {},
}
```

| Field | Type | Description |
| ----- | ---- | ----------- |
| `id` | `number` | **Unique and stable.** It is what gets stored in players' favourites: renumbering moves everyone's favourites onto other tracks |
| `title` / `artist` / `album` | `string` | Displayed metadata, truncated to 80 characters each |
| `duration` | `number` | Length in **seconds**, used by the progress bar. A wrong value does not prevent playback, it only skews the display |
| `url` | `string` | Address xsound will play. Required |
| `grad` | `{string, string}` | Cover art: two CSS colours. Optional: a colour is derived from the title if you omit it |

## 2. Group them into playlists

`tracks` holds `id` values from the list above.

```lua
    playlists = {
        { name = 'Night drive', tracks = { 1, 2 }, grad = { '#5e5ce6', '#ad6bff' } },
    },
```

A playlist whose tracks have all disappeared is not displayed at all, rather than shown empty.
`name` is truncated to 60 characters.

## 3. Restart

```
restart nash_phone
```

The library is read once at startup and cached, so a change needs a restart.

## Why a track does not appear

The server filters the list before the app ever sees it, so that a player can never tap a
track that will not play. A track is dropped when:

- it has **no `url`**;
- its address does not start with `http://` or `https://`;
- its address is longer than **512 characters**;
- its `id` is a **duplicate** of an earlier track: the first one wins, the second is ignored.

A track missing from the app is almost always a track whose address was copied incorrectly.

## The audio settings, and where they really live

{% hint style="danger" %}
`config/main.lua` declares a `Config.Music` block holding `distance`, `defaultVolume`,
`maxVolume` and `stopOnDeath`. `config/music.lua` is loaded **after** it and assigns
`Config.Music` outright, which **replaces** that block instead of completing it.

Editing those four keys in `config/main.lua` therefore has no effect. The server falls back to
its built-in values: `distance = 12.0`, `defaultVolume = 0.4`, `maxVolume = 1.0`,
`stopOnDeath = true`: which happen to be the same as the shipped ones, so nothing looks
broken until you actually change one.
{% endhint %}

To change them for real, put them in the **same table** as your tracks:

```lua
-- config/music.lua
Config.Music = {
    distance = 20.0,        -- listening range in metres (proximity audio)
    defaultVolume = 0.4,    -- starting volume (0.0 -> 1.0)
    maxVolume = 1.0,        -- ceiling allowed to the player
    stopOnDeath = true,     -- cut the music when the player dies

    tracks = { ... },
    playlists = { ... },
}
```

`stopOnDeath = true` is recommended: otherwise a body on the ground keeps broadcasting to the
whole block and nobody can stop it. The player always keeps the Stop button, in the Music app
and in Control Center; closing the phone does **not** stop the music, which is intentional.

## Private listening

Music comes out of the speaker by default: everyone nearby hears it. A player carrying the
earbuds item switches to private listening, where only they hear it. That is configured
separately in `config/main.lua`:

```lua
Config.Earbuds = {
    enabled = true,
    itemName = 'phone_earbuds',
    requireItem = true,
}
```

The resource creates no item. Create `phone_earbuds` in **your** inventory, or import
`sql/item_earbuds.sql` if your framework keeps items in the database. If the item does not
exist, the server prints a yellow warning at startup and nobody can pair earbuds.

## How to know it works

1. `restart nash_phone`, then open Music in game.
2. Your tracks are listed. If the library still shows the empty screen, either every track was
   filtered out or `Config.Music.tracks` was overwritten by another edit.
3. Press play and walk away: the sound follows you and fades past `distance` metres. Another
   player standing next to you hears it too: that is proximity audio working.
4. With `Config.Debug = true` in `config/main.lua`, run `/phoneconfig`. It prints the declared
   versus the playable count, which answers the "why is my track missing" question directly:

    ```
    Musique             12 titres jouables sur 14 declares, 2 listes de lecture
      -> 2 titre(s) ecarte(s) : adresse absente, non http(s), ou id en double.
    ```

5. `/phonecheck` flags the overwritten audio block explicitly:

    ```
    reglages audio de Config.Music perdus : config/music.lua ecrase config/main.lua.
    ```

6. `/phonedeps` tells you whether `xsound` is actually started.
