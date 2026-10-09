# Configuration Files Overview

NASH Phone is split into **14 configuration files**, each focused on one area. All of them
live in `nash_phone/config/`.

| File | Purpose |
| ---- | ------- |
| [`main.lua`](config-main.md) | Language, brand, mail, byCloud, phone item, numbers, video calls, prop and animations, voice, music, earbuds, camera, photo import |
| [`bridge.lua`](config-bridge.md) | Framework and inventory selection (`auto` by default) |
| [`maps.lua`](config-maps.md) | Maps app: world bounds, categories, points of interest |
| [`settings.lua`](config-settings.md) | The city name shown inside the phone |
| [`notifications.lua`](config-notifications.md) | Clearing a player's notifications on join / drop |
| [`apps.lua`](config-apps.md) | Enable, rename and pre-install each of the 33 applications |
| [`custom_apps.lua`](config-custom-apps.md) | Third-party applications added to the phone |
| [`music.lua`](config-music.md) | Music app library: tracks and playlists |
| [`social.lua`](config-social.md) | What players see of each other in the social apps, and the Cindr dating app |
| [`services.lua`](config-services.md) | Services app: the businesses citizens can call |
| [`setup.lua`](config-setup.md) | First-launch setup flow (hello, language, byCloud, Face ID, passcode) |
| [`upload.lua`](config-upload.md) | Image and video hosting: **server-side only, holds an API key** |
| [`turn.lua`](config-turn.md) | TURN relay for video calls and lives, recommended on a public server: **server-side only, holds an API token** |
| [`logs.lua`](config-logs.md) | Discord and Fivemanage logs, admin and public: **server-side only, holds webhooks** |

## Load order

`fxmanifest.lua` loads eleven of these files as **shared scripts**, in a fixed order:
`main.lua` creates the `Config` table, and every other file adds to it. Moving a file above
`main.lua`, or removing `main.lua`, breaks the ones that come after it.

`config/upload.lua`, `config/turn.lua` and `config/logs.lua` are the exceptions: they are
declared in `server_scripts` and must stay there. They hold an image-hosting API key, a TURN API
token and your log webhooks, and every shared script is downloaded into each connecting
player's cache.

## Editable files and escrow

NASH Phone is protected by FiveM Asset Escrow. The files of `config/` and
`server/bridge/custom.lua` (see [config/bridge.lua](config-bridge.md)) are the only Lua files you
can read and edit; every other Lua file of the resource is encrypted. Everything you can change
is described in these pages.

## Missing or malformed files

Most blocks have a fallback: a value of the wrong type, or a missing key, gives the shipped
default, and several blocks print a line in the server console when they had to ignore
something. A few missing blocks turn their feature off instead (`Config.VideoCall`,
`Config.Earbuds`, `Config.Services`, `Config.Logs`, `Config.Upload`), and some blocks of
`config/main.lua` have no fallback at all (`Config.OpenCommand`, `Config.Phone`, `Config.Prop`).
Keep every block of the shipped files and change values rather than deleting lines.

## Checking your configuration

There is no command that reads the configuration back. Start the server and read its console:
`nash_phone` reports what it found while it boots. Most of these lines are printed in French.

| Line in the server console | Meaning |
| -------------------------- | ------- |
| `[nash_phone] schéma DB prêt (... tables).` | The database tables were created or checked |
| `[nash_phone] framework détecté : ...` and `[nash_phone] inventaire détecté : ...` | The bridges in use, see [config/bridge.lua](config-bridge.md) |
| `[nash_phone] Langue « fr » chargee : ...` | One green line per language file loaded, see [config/main.lua](config-main.md#language) |
| `[nash_phone] locales/<code>.json est introuvable. ...` | A language set in `config/main.lua` has no file |
| `[nash_phone] Config.Upload is NOT configured -> the CAMERA IS DISABLED.` | Red block: no image host, see [config/upload.lua](config-upload.md) |
| `[nash_phone] video: no TURN relay configured, ...` | Yellow line: some players will get video calls without picture, see [config/turn.lua](config-turn.md) |
| `[nash_phone] video: Cloudflare TURN relay ready.` or `... request failed (HTTP <code>) ...` | Green or red line: the TURN relay works, or the request to Cloudflare failed, see [config/turn.lua](config-turn.md) |
| `[nash_phone] Config.UseItem = true but the item "phone" DOES NOT EXIST` | Red block, about fifteen seconds after startup: nobody can open the phone, see [Phone item](config-main.md#phone-item) |
| `[nash_phone] config/apps.lua : attention aux consequences ci-dessous.` | An app other parts of the phone rely on is turned off, see [config/apps.lua](config-apps.md) |

Everything else is checked in game: open the phone and the app concerned. Client-side messages,
such as a prop model that could not be loaded, appear in the player's own F8 console.
