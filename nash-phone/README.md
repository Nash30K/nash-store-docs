# nash_phone

NASH Phone is a premium smartphone for FiveM. It ships 33 applications, real calls, video
calls and text messages between players, seven social apps (six of them exchanging real
content between players), a camera that captures the actual game view, a map showing where
your friends really are, and dozens of per-player settings. Everything is available
in French and English, and each player picks their own language in the phone's settings.

ESX, QBCore and QBOX are detected automatically. A custom framework is wired in by filling a
single file (`server/bridge/custom.lua`), without touching anything else. The inventory is
chosen separately: ox_inventory, qs-inventory, qb-inventory (and its ps / lj forks), the
framework's own, or your own.

## Features

- 33 applications, from calls and messages to a bank wallet, a music player, a gallery, a
  services app for police, EMS, mechanics and taxis, and a set of games. A few are decorative
  on purpose (Browser, Weather, Health, Stocks): see the [Application Catalogue](apps/README.md)
- **Real calls between players**, with voice through pma-voice, speaker mode that lets nearby
  players join the conversation, conference calls of up to four people, and a call screen
  shared by every app that can place one
- **Video calls and InstaPic live streams:** the phone renders the game view into a canvas
  and streams it player to player. The image never passes through the server. A live host
  can bring up to three viewers on screen as guests
- **Seven social apps** with their own accounts: ChatApp, InstaPic, Snapz, BirdBy and TickTok
  exchange real content through the database (posts, stories and private messages, depending
  on the app), Cindr is a dating app with real matches and chats, and PlayTube has a real
  account but local content
- **A byCloud account**: it backs up the password keychain, so a player finds it again on
  another phone, and it opens its owner's mailbox, so stolen credentials have consequences
- **Face ID tied to the phone item:** each handset recognises the character who first used it
  (with an inventory that gives items a serial number, such as ox_inventory)
- **A first-launch setup**, like a new phone: language, appearance, byCloud account, Face ID
  and passcode
- **A camera that captures the game world**, with photos and videos saved to a gallery that
  every other app can read
- **A map showing friends' real positions**, computed server-side with an optional blur
- **Wireless earbuds** as an inventory item, for private music listening
- **Animations with the phone in hand**, on foot, as a driver and as a passenger
- **Discord and Fivemanage logs**, with staff logs and public roleplay feeds
- Per-player settings: wallpaper (built-in, or one of the player's own photos, framed with
  drag and zoom), lock screen, dark mode, ringtone, language, and a performance mode for
  lower-end machines
- Third-party applications can be added from your own resource, without touching the phone
- Runtime locales: drop a `locales/<lang>.json` file to add a language, no rebuild needed
- ESX / QBCore / QBOX / custom framework support through a single bridge file
- ox_inventory / qs-inventory / qb-inventory / framework inventory / custom inventory support
- 60 server exports and 73 client exports, most of them named after the most widespread
  convention, so existing scripts often work without modification (some of these names are
  neutral compatibility stubs: see the [Developer API](developer-api/README.md))

## Requirements

| Resource | Purpose |
| -------- | ------- |
| A **framework** | `es_extended` (ESX), `qb-core` (QBCore), `qbx_core` (QBOX), or any custom framework via the [bridge](compatibility/custom-framework.md) |
| `oxmysql` | Database access |
| `ox_lib` | Callbacks and fallback interface |
| **OneSync** | Must be enabled on the server |

The resource creates no inventory item. With `Config.UseItem = true` (the default), the `phone`
item must exist in your inventory, or nobody can open the phone. See
[Inventory items](installation/inventory-items.md).

Optional dependencies never block startup. The phone boots without them, and the table below
lists what each missing one costs.

| Resource | What you lose without it |
| -------- | ------------------------ |
| `pma-voice` | Voice in calls. The call still rings and displays, but stays silent. pma-voice is the only voice resource the phone drives |
| `screenshot-basic` | The profile pictures taken from the character's head (Snapz, ChatApp), and the camera's fallback when its viewfinder has no game image |
| `nash-phoneprop` | The 3D model held in hand. A fallback model takes over |
| `xsound` | Music playback from the Music app, and private listening with the earbuds |

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
