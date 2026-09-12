# Changelog

All notable changes to NASH Phone are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/) and the project adheres to
[Semantic Versioning](https://semver.org/).

---

## [Unreleased] - 2026-09-10

Photos can now add a picture from a link, and copy the link of a photo or video.

### Added

- **Add a photo from a link.** The **+** button of the Albums tab in Photos opens an
  **Add Photo by URL** alert. The player pastes a link, Discord included, and the picture lands
  in the camera roll. The player's game downloads it, checks it (JPEG, PNG, GIF or WebP,
  15 MB and 8192 pixels per side by default), re-encodes it like a camera photo, which also
  removes its EXIF metadata and GPS position, and uploads it to your Fivemanage storage. A
  Discord link is never stored as it is, and your server never downloads a player's link. The
  button only appears when an image host is configured. See
  [Photos](apps/README.md#media-and-capture).
- **`Config.Gallery.Import`** in `config/main.lua`: `Enabled`, `MaxMegabytes`, `MaxSidePx` and
  `AllowedHosts`. The shipped values work as they are; a Fivemanage account on a custom domain
  adds that domain to `AllowedHosts`. See
  [config/main.lua](config/config-main.md#adding-photos-by-link).
- **Recently Saved**, a collection in the Albums tab that gathers the photos added by link and
  the media received over AirDrop.
- **Copy** in the new **…** (More) menu of the full-screen viewer, and **Copy Photo** /
  **Copy Video** in the share sheet. The media's Fivemanage link goes to the clipboard; pasted
  into Discord, the link of a photo displays as the picture. Such a link is public, and how long
  it lasts depends on the retention set on your Fivemanage account.
- **Duplicate** in the same **…** menu: a second copy of the media in the camera roll, next to
  the original, with nothing uploaded again.
- **Snapz takes snaps with the Camera app's camera.** Same lens, same controls (mouse to aim,
  wheel to zoom, Enter, Q, Alt, Escape) and the same key hints. The snap is taken inside the
  viewfinder, with the lens tint and night mode applied. See
  [Snapz](apps/README.md#taking-a-snap).

### Fixed

- **Snapz snaps.** Taking a snap used to screenshot the whole screen, which hides the phone and
  closes the app: the snap never reached the caption screen.
- **Camera: thumbnails.** The thumbnail of each photo was uploaded but not saved, so the Photos
  grid loaded every photo at full size. It is now saved with the photo.
- **Camera: phone put away and taken out quickly.** The viewfinder came back without its camera
  (Enter, Alt and Q did nothing). It now restarts with the phone.
- **Camera: leaving the camera no longer reloads the whole phone from the server**, and no longer
  wakes a screen the player had just turned off with the side button.
- **Camera: combat, vehicle and chat controls are neutralized while the viewfinder is on**, as
  they already were with the phone open.
- **Accepting a photo or video received over AirDrop.** The server failed to save it, so the
  media never reached the recipient's camera roll. It now lands there, and in Recently Saved.
- **The clipboard in game.** Notes, InstaPic, TickTok, the contact card and Passwords now copy
  through one method the game's browser accepts. A copy is only confirmed when the text really
  reached the clipboard (the contact card and Passwords used to say "copied" when nothing was),
  and copying no longer leaves the keyboard captured: the player can walk again straight after.

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
- First-run setup flow, shown once per character, and again on a **different handset**: each
  device carries its own serial, so a new phone asks for the setup and the byCloud account is
  reopened with its password. Storing the phone in a trunk changes nothing, and messages,
  contacts and the phone number stay with the character.

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
