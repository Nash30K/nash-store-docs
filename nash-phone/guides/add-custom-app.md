# Add a third-party application

Ship your own application inside the phone from your own resource, without touching the
phone's code.

## Prerequisites

- A normal FiveM resource of your own
- `nash_phone` started **before** it
- The `nash_exemple_app` resource, shipped alongside: it is the complete model to copy: three
  Lua files and one HTML page, and it exercises everything below

## In thirty seconds

```lua
-- client.lua of YOUR resource
CreateThread(function()
    while GetResourceState('nash_phone') ~= 'started' do Wait(200) end
    exports['nash_phone']:AddCustomApp({
        identifier = 'my_bank',
        name = 'My Bank',
        ui = GetCurrentResourceName() .. '/ui/index.html',
        icon = 'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png',
    })
end)
```

```html
<!-- ui/index.html -->
<script src="https://cfx-nui-nash_phone/custom_app_bridge/nash-bridge.js"></script>
```

That is all. Your interface now has `fetchNui`, `onNuiEvent`, `onSettingsChange`, `createCall`
and `components`.

## The three traps that cost the most time

{% hint style="danger" %}
**The `ui` path starts with the name of YOUR resource.** Writing `ui/index.html` would point
the iframe at the phone itself: blank page, no error. The phone actually refuses a path with
no resource name and tells you in the console:

```
[nash_phone] app personnalisée refusée (my_resource) : ui = 'ui/index.html' n'a pas de nom de ressource ; attendu 'my_resource/ui/index.html'
```

**Declare your files in the `files{}` block of your `fxmanifest.lua`.** A file that is not
declared is not served to the player. Same symptom: blank page, no message.

**Do not load your data from `onUse`.** The app opens before your interface has finished
loading, so the message would arrive into the void. Your interface asks, with `fetchNui`, when
it is ready.
{% endhint %}

## 1. The manifest

```lua
-- fxmanifest.lua of YOUR resource
fx_version 'cerulean'
game 'gta5'
lua54 'yes'

client_scripts { 'client.lua' }
server_scripts { 'server.lua' }

files {
    'ui/index.html',
    'ui/**',
}
```

## 2. Declare the app

Two ways to add an app.

<details>

<summary>From your own resource: with the AddCustomApp export</summary>

This is what you need when your app has an interface, because its HTML has to live in **your**
resource. Call it **client side**, once `nash_phone` is started.

```lua
CreateThread(function()
    while GetResourceState('nash_phone') ~= 'started' do Wait(200) end

    exports['nash_phone']:AddCustomApp({
        identifier = 'nash_example',
        name = 'Example',
        description = 'Custom application model for NASH Phone.',
        developer = 'NASH',
        ui = GetCurrentResourceName() .. '/ui/index.html',
        icon = 'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png',
        defaultApp = true,
        size = 240,
        onUse = function()
            print("[example] the player just opened the app")
        end,
    })
end)
```

The app is removed automatically when your resource stops: otherwise its icon would stay on
the home screen and opening it would load an iframe pointing at a file that is no longer
served.

```lua
exports['nash_phone']:RemoveCustomApp('nash_example')
```

</details>

<details>

<summary>In the phone's config: Config.CustomApps</summary>

Simpler, and the only place where `onServerUse` works (see below). Suitable for an app with no
interface of its own.

```lua
-- config/custom_apps.lua
Config.CustomApps = {
    example = {
        identifier = 'example',
        name = 'Example',
        description = 'An example app, with no interface.',
        developer = 'NASH',
        defaultApp = true,
        size = 128,
        onUse = function()
            exports['nash_phone']:SendNotification({
                app = 'example',
                title = 'Example',
                content = 'The app was opened.',
            })
        end,
        onServerUse = function(source)
            print(('[nash_phone] player %s opened the example app'):format(source))
        end,
    },
}
```

The table key doubles as a fallback identifier, so `Config.CustomApps.my_app = { name = ... }`
works without an explicit `identifier`.

</details>

## 3. The fields

| Field | Type | Description |
| ----- | ---- | ----------- |
| `identifier` | `string` | **Required.** Unique id, never shown to the player. Two apps cannot share one |
| `name` | `string` | Name under the icon. Falls back to `identifier` |
| `description` | `string` | Description, visible in the App Store |
| `developer` | `string` | Author name, visible in the App Store |
| `icon` | `string` | Icon URL. For an image in your resource: `'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png'` |
| `ui` | `string` | Path to your interface's HTML, **prefixed with your resource name**: `'my-resource/ui/index.html'`. Without it the app only calls `onUse` when tapped, and opens nothing |
| `defaultApp` | `boolean` | `true` = placed on the home screen at first boot. `false` = downloaded from the App Store. Default `true` |
| `game` | `boolean` | `true` = filed under the App Store's Games category |
| `size` | `number` | Size advertised in the App Store, in KB. Purely cosmetic |
| `images` | `string[]` | Screenshot URLs shown on the App Store sheet |
| `price` | `number` | In-game price to download it. `0` = free |
| `landscape` | `boolean` | `true` = the app displays in landscape |
| `keepOpen` | `boolean` | `true` = the phone does **not** close when the player opens the app. Reserve this for apps that take over the screen, such as a camera |
| `onUse` | `function` | Called **client side** when the app is opened |
| `onServerUse` | `function` | Called **server side** when the app is opened. Receives the player's `source` |

{% hint style="warning" %}
**`onServerUse` only works for apps declared in `Config.CustomApps`.** A Lua function does not
cross the network: an app declared at runtime by `AddCustomApp` from a *client* script cannot
deliver its `onServerUse` to the server, because the function only exists in the client
context.

For those apps, listen to the public event from your own server script instead:

```lua
-- server.lua of YOUR resource
AddEventHandler('nash_phone:customAppUsed', function(src, identifier)
    if identifier ~= 'nash_example' then return end
    print(('[example] player %s opened the app'):format(src))
end)
```
{% endhint %}

## 4. Talk to your interface

Your interface asks, using `fetchNui`. You answer with a normal `RegisterNUICallback` in your
own resource:

```lua
-- client.lua of YOUR resource
RegisterNUICallback('getInfos', function(_, cb)
    local ped = PlayerPedId()
    cb({
        name = GetPlayerName(PlayerId()),
        health = math.floor(GetEntityHealth(ped) / 2),
    })
end)
```

```js
// ui/index.html
fetchNui('getInfos').then((d) => {
  document.getElementById('name').textContent = d.name
})
```

To push into your interface without being asked, from the client:

```lua
exports['nash_phone']:SendCustomAppMessage('nash_example', {
    action = 'tick',
    data = { hour = GetClockHours(), minute = GetClockMinutes() },
})
```

or from the server, when the data is authoritative (a balance, an inventory) and you do not
want the client computing it:

```lua
exports['nash_phone']:SendCustomAppMessage(source, 'nash_example', {
    action = 'tick',
    data = { balance = 1200 },
})
```

Either way it comes out in the iframe as:

```js
onNuiEvent('tick', (d) => {
  console.log(d.hour, d.minute)
})
```

## 5. The bridge API

Loaded by that one `<script>` tag, **before** your own script. It is served by the phone, so
your app follows its updates without copying anything. To freeze it instead, copy
`nash-bridge.js` into your resource and change the path.

| Function | State |
| -------- | ----- |
| `fetchNui(event, data, scriptName?)` | available |
| `onNuiEvent(action, handler)` | available |
| `onSettingsChange(handler)` | available |
| `createCall(options)` | available |
| `resourceName`, `appName`, `settings` | available |

| Component | State |
| --------- | ----- |
| `setPopUp` | available |
| `setContextMenu` | available |
| `setContactSelector` | available |
| `setColorPicker` | available |
| `setEmojiPickerVisible` | available |
| `setGallery` | available |
| `setFullscreenImage` | available |
| `setHomeIndicatorVisible` | available |
| `saveToGallery` | available |
| `uploadMedia` | not wired yet |
| `setShareComponent` | not wired yet |
| `setGifPickerVisible` | not wired yet |
| `createGameRender` | not wired yet |
| `GameMap` | not wired yet |

Nothing on that list crashes your app. What is not wired yet returns a neutral value **and
writes a console warning**, so you know which one you are missing:

```
[nash-phone] composant « uploadMedia » pas encore disponible.
```

Components are painted **by the phone, above your page**, so they cover the whole screen the
way they do on iOS.

```js
components.setPopUp({
  title: 'An alert',
  description: 'Drawn by the phone, not by your app.',
  buttons: [
    { title: 'Cancel', color: 'red' },
    { title: 'OK', cb: () => console.log('OK') },
  ],
})
```

## 6. Theme and layout

The bridge sets `data-theme="dark"` or `"light"` on `<html>` and updates it when the player
switches mode. Hook your CSS onto it: an app that stays dark on a light phone stands out
immediately.

```css
:root        { --bg: #000; --ink: #fff; }
html[data-theme='light'] { --bg: #f2f2f7; --ink: #000; }
```

The status bar and the Dynamic Island are painted **by the phone, on top of you**. Leave
**62 px of padding at the top** and **34 px at the bottom** for the home indicator, as the
example app does.

```css
body { padding: 62px 20px 34px; }
```

## 7. Global limits

```lua
-- config/custom_apps.lua
Config.CustomAppsSettings = {
    max = 40,      -- maximum custom apps accepted at once
    debug = false, -- print every add and every removal in the console
}
```

`max` is a guard rail: a buggy resource calling `AddCustomApp` in a loop cannot flood the
player's home screen. Past the ceiling the phone refuses and says so:

```
[nash_phone] app personnalisée « x » refusée : plafond de 40 atteint (Config.CustomAppsSettings.max)
```

Turn `debug = true` on when an app does not show up and you want to know whether it was even
declared.

## Applications written for another phone

The field and function names above are deliberately the ones used by the most widespread
custom-app convention (LB Phone): an app written for that phone declares here without any
change to its table or to its interface code.

The only obstacle is the resource name it calls in its own Lua.

- **If you can open its file**: replace that name with `nash_phone`. One line. This is the
  recommended route: no extra resource, no name collision.
- **If it is escrow-protected**: use the `nash_lbcompat` resource, shipped separately, which
  answers to the expected name and forwards the calls. Read its `README.md` first: it has to be
  **renamed** to the name the app calls (FiveM resolves `exports['a-name']` by folder name), it
  must start **after** `nash_phone`, and it must never run alongside the original resource -
  two resources cannot share a name, and the server would start one of them at random.

An app is only as portable as the functions it uses. The table above, plus the
[Developer API](../developer-api/README.md) for the Lua side, say exactly what you can rely on.

## How to know it works

1. Start your resource after `nash_phone`. The icon appears on the home screen (with
   `defaultApp = true`) or in the App Store.
2. Open the app. Your HTML renders inside the phone frame, with the status bar over it.
3. If the page is blank, in this order: check that `ui` starts with your resource name, that
   `files{}` declares your HTML, and that the file path is right. None of those three produces
   an error message on their own.
4. Set `Config.CustomAppsSettings.debug = true` and restart: every add and removal is printed,
   which tells you whether the declaration ever reached the phone.
5. Switch the phone to light mode from Control Center. Your app should follow.
6. Stop your resource with the phone open: the icon disappears on its own.
