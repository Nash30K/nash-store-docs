# Installation Overview

A six-step install. Each step has its own page.

## Quick start

1. [Install the dependencies](requirements.md) and check your framework / inventory
2. Drop `nash_phone` (and `nash-phoneprop`) into your `resources/` folder
3. [Let the database build itself](database.md): no SQL file to import
4. [Declare the inventory items](inventory-items.md): `phone`, and `phone_earbuds` if you want private listening
5. [Load the 3D model resource](streamed-prop.md) before `nash_phone`
6. [Fill in `config/upload.lua`](image-hosting.md): mandatory for the camera

Then go to [First Launch](first-launch.md) to verify the install and learn the diagnostic
commands.

## Folder layout after install

```
resources/
├── nash_phone/        # main resource (required)
└── nash-phoneprop/    # the phone model held in hand (optional, but start it first)
```

Inside `nash_phone`:

```
nash_phone/
├── fxmanifest.lua
├── config/            # 12 configuration files, all commented
├── client/  server/  shared/
├── locales/           # fr.json, en.json (read at runtime)
├── sql/               # reference schema + the two item files (nothing to import)
├── assets/prop/       # the coloured shells of the 3D model
├── custom_app_bridge/ # bridge for third-party applications
├── web/build/         # the compiled interface
└── README.md  LANGUES.md
```

## Load order

{% hint style="warning" %}
`nash-phoneprop` must start **before** `nash_phone`. If the model is not registered yet when
the phone boots, the phone silently falls back to a GTA prop (`Config.Prop.fallback`) and
players hold the wrong device.
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
| Import the SQL file | The schema is created and migrated at boot by `server/db/schema.lua`. See [Database](database.md) |

## Configuration files

All of them live in `nash_phone/config/` and are loaded as shared scripts, **except**
`config/upload.lua`, which is a server script and must stay one.

| File | What it controls |
|---|---|
| `main.lua` | Locale, open key, item, phone numbers, prop, voice, camera, brand, debug |
| `bridge.lua` | Framework and inventory selection (`auto` by default) |
| `apps.lua` | Which applications exist and which are pre-installed |
| `settings.lua` | Default per-player settings |
| `social.lua` | Social networks: story audience, notification purge, map |
| `notifications.lua` | Notification lifetime and purge |
| `maps.lua` | Places shown in the Maps app, in GTA coordinates |
| `music.lua` | Music library |
| `services.lua` | Jobs reachable from the Services app |
| `setup.lua` | The first-open setup wizard |
| `custom_apps.lua` | Your own applications |
| `upload.lua` | **Image hosting: server script, contains a secret** |
