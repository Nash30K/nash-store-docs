# nash_phone

**Latest release: v0.1.0.** See the [Changelog](changelog.md) for what's new.

NASH Phone is a premium smartphone for FiveM. It ships around thirty applications, real
calls and text messages between players, seven working social networks, a camera that
captures the actual game view, a map showing where your friends really are, and roughly a
hundred per-player settings. Everything is available in French and English, and each player
picks their own language in the phone's settings.

ESX, QBCore and QBOX are detected automatically. A custom framework is wired in by filling a
single file, without touching anything else. The inventory is chosen separately: ox_inventory,
qb-inventory, the framework's own, or your own.

## Features

- Around thirty applications, from calls and messages to a bank wallet, a music player, a
  gallery and a set of games
- **Real calls between players**, with voice through pma-voice, speaker mode, and a call
  screen shared by every app that can place one
- **Video calls and InstaPic live streams:** the phone renders the game view into a canvas
  and streams it player to player. The image never passes through the server
- **Seven social networks** with real accounts, followers, posts, stories and private
  messages, all backed by the database
- **A camera that captures the game world**, with photos and videos saved to a gallery that
  every other app can read
- **A map showing friends' real positions**, computed server-side with an optional blur
- Per-player settings: wallpaper, lock screen, dark mode, ringtone, language, and a
  performance mode for lower-end machines
- Third-party applications can be added from your own resource, without touching the phone
- Runtime locales: drop a `locales/<lang>.json` file to add a language, no rebuild needed
- ESX / QBCore / QBOX / custom framework support through a single bridge file
- ox_inventory / qb-inventory / framework inventory / custom inventory support
- Around sixty exports, most of them named after the most widespread convention, so existing
  scripts often work without modification

## Requirements

| Resource | Purpose |
| -------- | ------- |
| A **framework** | `es_extended` (ESX), `qb-core` (QBCore), `qbx_core` (QBOX), or any custom framework via the [bridge](compatibility/custom-framework.md) |
| `oxmysql` | Database access |
| `ox_lib` | Callbacks and fallback interface |
| **OneSync** | Must be enabled on the server |

Optional dependencies never block startup. The phone boots without them and tells you in the
console what it lost.

| Resource | What you lose without it |
| -------- | ------------------------ |
| `pma-voice` | Voice in calls. The call still rings and displays, but stays silent |
| `screenshot-basic` | The camera can no longer save anything |
| `nash-phoneprop` | The 3D model held in hand. A fallback model takes over |
| `xsound` | Music playback from the Music app |

{% hint style="warning" %}
`config/upload.lua` ships with an **empty** image-hosting key. Fill it in before going live,
or the camera, the gallery and profile pictures will not be able to save anything. See
[Image hosting](installation/image-hosting.md).
{% endhint %}

## Links

- [Installation](installation/README.md)
- [Compatibility](compatibility/README.md)
- [Configuration Files](config/README.md)
- [Applications](apps/README.md)
- [FAQ](faq.md)
- [Common Errors](common-errors.md)
- [Guides](guides/README.md)
- [Developer API](developer-api/README.md)
