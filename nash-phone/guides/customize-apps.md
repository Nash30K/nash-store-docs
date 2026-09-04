# Choose which apps are installed

Decide which of the shipped applications exist on your server, which are preinstalled on the
home screen, which have to be downloaded from the App Store, and what each one is called.

## Prerequisites

- Filesystem access to the `nash_phone` resource folder
- Everything happens in a single file: `config/apps.lua`

## The three settings

`Config.Apps` maps an application id to a table with three optional fields.

| Field | Type | Effect |
| ----- | ---- | ------ |
| `enabled` | `boolean` | `false` makes the app **disappear completely**: no icon, absent from the App Store, from the library, from the search and from Settings. It is also removed from the home screen of players who already had it |
| `name` | `string` | Displayed name, replacing the original one. It replaces the *translation*, so the same name shows in French and in English: which is the point ("Chicago Taxi" has no business being translated) |
| `store` | `boolean` | `true` = the app is **not** on the phone at first, the player has to download it from the App Store. `false` = preinstalled, placed on the home screen at first boot |

Anything you leave out keeps its shipped value. An application missing from the table works
normally, so you can delete the lines you do not care about.

`store` counts in **both directions**: it takes an app out of the App Store as readily as it
puts one in.

## The shipped applications

```lua
-- config/apps.lua
Config.Apps = {
    -- Dock: the four applications always within reach
    phone     = { enabled = true },
    messages  = { enabled = true },
    camera    = { enabled = true },
    wallet    = { enabled = true },

    -- Preinstalled, placed on the home screen at first boot
    contacts  = { enabled = true },
    gallery   = { enabled = true },
    mail      = { enabled = true },
    notes     = { enabled = true },
    maps      = { enabled = true },
    weather   = { enabled = true },
    clock     = { enabled = true },
    calendar  = { enabled = true },
    facetime  = { enabled = true },
    music     = { enabled = true },
    safari    = { enabled = true },
    health    = { enabled = true },
    reminders = { enabled = true },
    passwords = { enabled = true },
    services  = { enabled = true },
    appstore  = { enabled = true },
    settings  = { enabled = true },

    -- To be downloaded from the App Store
    calculator = { enabled = true, store = true },
    compass    = { enabled = true, store = true },
    stocks     = { enabled = true, store = true },
    snake      = { enabled = true, store = true },
    tictactoe  = { enabled = true, store = true },

    -- Social networks
    ticktok  = { enabled = true, store = true },
    instapic = { enabled = true, store = true },
    playtube = { enabled = true, store = true },
    snapz    = { enabled = true, store = true },
    chatapp  = { enabled = true, store = true },
    birdby   = { enabled = true, store = true },
}
```

{% hint style="info" %}
`safari` is a **decorative** browser. It renders a start page with the favourites from
`Config.Brand.site` and goes nowhere: the embedded browser the phone runs in has no network
access at all. Keep it or disable it, but do not expect it to load a real website.
{% endhint %}

## 1. Preinstall an App Store app

Give it `store = false`:

```lua
compass = { enabled = true, store = false },
```

Players who already play on the server get it added to the last page of their home screen on
their next connection. Only apps you **explicitly** marked `store = false` are placed that
way: the rest is the player's own layout, and the phone does not touch it.

## 2. Move a preinstalled app into the App Store

Give it `store = true`:

```lua
music = { enabled = true, store = true },
```

Players who already have it on their home screen keep it. New characters have to download it.

## 3. Rename an app

```lua
maps     = { enabled = true, name = 'Navigator' },
services = { enabled = true, name = 'Los Santos Services' },
snapz    = { enabled = true, store = true, name = 'SnapCity' },
```

The name replaces the translated one, so it is identical in every language.

## 4. Remove an app entirely

```lua
stocks = { enabled = false },
snake  = { enabled = false },
```

The app vanishes from the home screen, the App Store, the library, the search and Settings -
including for players who already had it.

{% hint style="danger" %}
**Four applications make the others usable.** Disabling them is not forbidden: it is your
server: but know what you lose:

| App | What breaks |
| --- | ----------- |
| `appstore` | No app marked `store = true` can be installed any more. They become permanently unreachable |
| `settings` | The player can no longer change wallpaper, ringtone or language, nor manage their account |
| `phone` | No calls, and the call log becomes unreachable |
| `messages` | No SMS. Other scripts can still send them, the player will not see them |

The phone does not stop you. It prints a warning once at startup.
{% endhint %}

The warning looks like this:

```
========================================================================
[nash_phone] config/apps.lua : attention aux consequences ci-dessous.
[nash_phone]    appstore desactive -> 11 app(s) marquee(s) 'store' sont inaccessibles.
[nash_phone]    settings desactive -> plus de fond d ecran, de sonnerie ni de langue.
========================================================================
```

## Adding your own applications

`Config.Apps` only governs the applications shipped with the phone. To add one of your own,
see [Add a third-party application](add-custom-app.md).

## How to know it works

1. `restart nash_phone`, then open the phone in game.
2. A disabled app must be gone from **four** places, not one: home screen, App Store, app
   library (swipe left past the last page) and the Settings app list.
3. A renamed app shows its new name under the icon and in the search results.
4. With `Config.Debug = true` in `config/main.lua`, run `/phoneconfig`. It prints one summary
   line built from the configuration the server **actually read**, which immediately exposes a
   malformed block that was silently ignored:

    ```
    Applications        32 reglees, 2 desactivees, 11 a telecharger
      -> « stocks » desactivee.
      -> « appstore » desactivee : plus aucune app a telecharger n est installable.
    ```

5. `/phonecheck` reports the same critical disables as a problem rather than as information.

{% hint style="warning" %}
Put `Config.Debug` back to `false` afterwards. The debug commands write to the database and
push events on the player's behalf.
{% endhint %}
