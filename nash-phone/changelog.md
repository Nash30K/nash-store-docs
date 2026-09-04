# Changelog

All notable changes to NASH Phone are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/) and the project adheres to
[Semantic Versioning](https://semver.org/).

---

## [0.1.0] - Initial release

First public release of NASH Phone (NphoneV1).

### Phone shell

- Around thirty applications, a home screen with pages, folders, widgets and an app library,
  a lock screen with configurable clock and widgets, a notification centre, a control centre,
  a multitasking switcher and a Spotlight search.
- **Around a hundred per-player settings**, stored server-side and restored on reconnect:
  wallpaper, lock screen layout, dark mode, ringtone, language, per-app notification
  switches, phone size and position, and a performance mode.
- **French and English**, chosen per player. Locale files are read at runtime from
  `locales/`, so a new language needs no rebuild. See
  [Add a language](guides/add-language.md).
- First-run setup flow, shown once per character.

### Communication

- **Real calls between players**, with voice routed through `pma-voice` on a dedicated
  channel, speaker mode that adds nearby players, and a single call screen shared by every
  app able to place a call.
- **Video calls.** The phone renders the game view into a canvas and streams it directly from
  one player to another. The server only relays the connection handshake, never the image.
  Up to four participants, split vertically on screen.
- Text messages, contacts with photos, a mail application with a real per-character address,
  and AirDrop between nearby players.

### Social

- Seven social applications backed by shared database tables: accounts with their own
  password, followers, posts, likes, comments, stories and private messages.
- **A friend map** in Snapz showing real in-world positions, computed server-side with an
  optional blur and mutual-consent sharing.
- **InstaPic live streams.** A player broadcasts their game view to the accounts that follow
  them, can invite a viewer on screen, and the screen splits between host and guest. Launch
  notifications are configurable per stream: everyone, followers only, or nobody.

### Media and tools

- A camera that captures the game world, with photos and videos written to a gallery every
  other application can read.
- Music player driven by a configurable library, a weather app, a map with configurable
  locations, a compass reading the game camera, notes, reminders, calendar, a password
  manager and a small set of games.

### Integration

- ESX, QBCore and QBOX detected automatically; any other framework is wired in through a
  single bridge file. The inventory is chosen separately.
- Around sixty exports, client and server, most named after the most widespread convention so
  that existing third-party scripts work unchanged. See the
  [Developer API](developer-api/README.md).
- Third-party applications can be added from your own resource, with no change to the phone.
  See [Add a custom app](guides/add-custom-app.md).
- The database is created on first start, and future updates add their own columns and
  indexes without manual work.
