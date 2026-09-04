# Configuration Files Overview

NASH Phone is split into **12 configuration files**, each focused on one area. All of them
live in `nash_phone/config/`.

| File | Purpose |
| ---- | ------- |
| [`main.lua`](config-main.md) | Language, brand, phone item, numbers, calls, camera, prop, earbuds |
| [`bridge.lua`](config-bridge.md) | Framework and inventory selection (`auto` by default) |
| [`maps.lua`](config-maps.md) | Maps app: world bounds, categories, points of interest |
| [`settings.lua`](config-settings.md) | The city name shown inside the phone |
| [`notifications.lua`](config-notifications.md) | Clearing a player's notifications on join / drop |
| [`apps.lua`](config-apps.md) | Enable, rename and pre-install each of the 32 applications |
| [`custom_apps.lua`](config-custom-apps.md) | Third-party applications added to the phone |
| [`music.lua`](config-music.md) | Music app library: tracks and playlists |
| [`social.lua`](config-social.md) | What players see of each other in the social apps |
| [`services.lua`](config-services.md) | Services app: the businesses citizens can call |
| [`setup.lua`](config-setup.md) | First-launch setup flow (hello, language, byCloud, Face ID, passcode) |
| [`upload.lua`](config-upload.md) | Image and video hosting: **server-side only, holds an API key** |

## Load order

`fxmanifest.lua` loads eleven of these files as **shared scripts**, in a fixed order:
`main.lua` creates the `Config` table, and every other file adds to it. Moving a file above
`main.lua`, or removing `main.lua`, breaks the ones that come after it.

`config/upload.lua` is the exception: it is declared in `server_scripts` and must stay there.
It holds an image-hosting API key, and every shared script is downloaded into each connecting
player's cache.

{% hint style="warning" %}
`config/music.lua` assigns `Config.Music` as a whole, and it is loaded **after**
`config/main.lua`. The audio block written in `main.lua` (`distance`, `defaultVolume`,
`maxVolume`, `stopOnDeath`) is therefore replaced and never read. Editing those four values in
`main.lua` changes nothing. See [config/music.lua](config-music.md) and the
[Music](config-main.md#music-proximity-audio) section of `config/main.lua`.
{% endhint %}

## Missing or malformed files

Every block has a written fallback in the Lua code. A deleted file, a `nil` table or a value of
the wrong type falls back to the shipped default instead of erroring, and most of these
fallbacks print a line in the server console. The resource is designed to boot without any of
its configuration files.

## Checking your configuration in game

With `Config.Debug = true` (see [config/main.lua](config-main.md#test-commands)), two commands
read back what the server actually resolved:

| Command | What it prints |
| ------- | -------------- |
| `/phoneconfig` | The configuration as the server read it: language, item, phone numbers, NearShare, speaker, camera, apps turned off |
| `/phonecheck` | A short verdict: missing dependencies, missing item, no upload host, disabled apps others depend on, non-square map bounds, phone numbers too long for the column |

Leave `Config.Debug` at `false` on a live server.
