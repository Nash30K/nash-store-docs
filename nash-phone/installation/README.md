# Installation Overview

A six-step install. Each step has its own page.

## Quick start

1. [Install the dependencies](requirements.md) and check your framework / inventory
2. Drop `nash_phone` (and `nash-phoneprop`) into your `resources/` folder
3. [Let the database build itself](database.md): no SQL file to import
4. [Declare the inventory items](inventory-items.md): `phone`, and `phone_earbuds` if you want private listening
5. [Load the 3D model resource](streamed-prop.md) before `nash_phone`
6. [Fill in `config/upload.lua`](image-hosting.md): mandatory for the camera

On a public server, also fill in [`config/turn.lua`](../config/config-turn.md): without a TURN
relay, some players get video calls with sound but no picture.

Then go to [First Launch](first-launch.md) to verify the install: what the server console
prints at startup, and the checks to run in game.

## Folder layout after install

```
resources/
├── nash_phone/        # main resource (required)
└── nash-phoneprop/    # the phone model held in hand (optional, but start it first)
```

{% hint style="warning" %}
Keep the folder name `nash_phone`. Other resources call it as `exports['nash_phone']`, and the
interface loads its own files from `cfx-nui-nash_phone/` (Services icons, the bridge used by
third-party applications). A renamed folder breaks both.
{% endhint %}

Inside `nash_phone`:

```
nash_phone/
├── fxmanifest.lua
├── config/            # 14 configuration files, commented: this is what you edit
├── client/  server/  shared/   # encrypted, except server/bridge/custom.lua
├── locales/           # fr.json, en.json (read at runtime)
├── INSTALL/           # ready-to-paste items (ox_inventory, qb-core, qs-inventory) + sql/
├── assets/prop/       # the coloured shells of the 3D model
├── service_icons/     # your PNG icons for the Services app
├── custom_app_bridge/ # bridge for third-party applications
├── web/build/         # the compiled interface
└── README.md  LANGUAGES.md
```

## What you can edit

The resource is protected by FiveM **Asset Escrow**. Among its Lua files, only these stay
readable and editable:

| Where | What for |
|---|---|
| `config/*.lua` | Every setting of the phone |
| `server/bridge/custom.lua` | Wiring a custom framework or a custom inventory. See [Custom Framework](../compatibility/custom-framework.md) |

Every other Lua file is encrypted. The interface ships compiled in `web/build/`, without its
sources. The JSON files of `locales/` (see [Add a language](../guides/add-language.md)), the
`INSTALL/sql/` files and the PNG files of `service_icons/` stay plain files you can open and edit.

{% hint style="info" %}
Asset Escrow only runs the resource on a server whose license key belongs to the Cfx.re account
that owns the asset (it is granted to that account through Keymaster when you buy it).
{% endhint %}

## Load order

{% hint style="warning" %}
`nash-phoneprop` must start **before** `nash_phone`. If the model is not available when a
player takes the phone out, the phone falls back to a GTA prop (`Config.Prop.fallback`), prints
one line in that player's F8 console, and players hold the wrong device.
{% endhint %}

```cfg
ensure oxmysql
ensure ox_lib
ensure ox_inventory        # or your inventory
ensure es_extended         # or qb-core / qbx_core
ensure pma-voice           # optional
ensure screenshot-basic    # optional
ensure xsound              # optional
ensure nash-phoneprop
ensure nash_phone
```

OneSync must be enabled on the server. It is declared as a hard dependency in
`fxmanifest.lua` (`dependencies { '/onesync', 'ox_lib', 'oxmysql' }`), so `nash_phone` will
refuse to start without it.

## What the resource never does for you

| Task | Why it is on your side |
|---|---|
| Create the inventory items | The resource declares no item. Your inventory owns the item list. See [Inventory items](inventory-items.md) |
| Provide an image host | Photos are stored as **links**. You need your own hosting token. See [Image hosting](image-hosting.md) |
| Import the SQL file | Nothing to do: the resource creates and migrates its tables at every boot. See [Database](database.md) |

## Configuration files

All of them live in `nash_phone/config/` and are loaded as shared scripts, **except**
`config/upload.lua`, `config/turn.lua` and `config/logs.lua`, which are server scripts and must
stay that way: they hold secrets (API tokens, webhook links).

| File | What it controls |
|---|---|
| `main.lua` | Locale, open key, item, phone numbers, prop, animations, voice, music audio, earbuds, camera, photo import, brand, mail domain, byCloud, video calls |
| `bridge.lua` | Framework and inventory selection (`auto` by default) |
| `apps.lua` | Which applications exist, which are pre-installed, and their names |
| `settings.lua` | The city name and region shown in the phone (`Config.City`) |
| `social.lua` | Social networks: story audience, notification purge, Snapz map, InstaPic lives, Cindr |
| `notifications.lua` | Purge of a player's phone notifications when they connect or disconnect |
| `maps.lua` | Places shown in the Maps app, in GTA coordinates |
| `music.lua` | Music library |
| `services.lua` | Jobs reachable from the Services app |
| `setup.lua` | The first-open setup wizard |
| `custom_apps.lua` | Your own applications |
| `upload.lua` | **Image hosting: server script, contains a secret** |
| `turn.lua` | **TURN relay for video calls, recommended on a public server: server script, contains a secret** |
| `logs.lua` | **Discord and Fivemanage logs: server script, contains webhook links** |
