# config/custom_apps.lua

The registry of third-party applications: yours, or anyone else's.

File: `config/custom_apps.lua`, loaded as a shared script.

There are two ways to add an app, and the choice depends on one thing only: whether it has an
interface.

| Way | When to use it |
| --- | -------------- |
| **Here**, in `Config.CustomApps` | The simplest. You write the app in this file and it appears on every phone on the server |
| **From your own resource**, with the `AddCustomApp` export | Required if your app has an interface, because its HTML has to live in **your** resource |

A working reference resource, `nash_exemple_app`, ships alongside the phone. It is three Lua
files and one HTML page, and it exercises everything described below.

## Declaring from your own resource

```lua
-- client.lua of YOUR resource
CreateThread(function()
    while GetResourceState('nash_phone') ~= 'started' do Wait(200) end
    exports['nash_phone']:AddCustomApp({
        identifier = 'ma_banque',
        name = 'Ma Banque',
        ui = GetCurrentResourceName() .. '/ui/index.html',
        icon = 'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png',
    })
end)
```

```html
<!-- ui/index.html -->
<script src="https://cfx-nui-nash_phone/custom_app_bridge/nash-bridge.js"></script>
```

That is the whole setup. Your interface now has `fetchNui`, `onNuiEvent`, `onSettingsChange`,
`createCall` and `components`.

Call `AddCustomApp` on the **client**, once the player is loaded. Apps registered by a
third-party resource are removed automatically when that resource stops, so an icon never
survives a `stop` and opens an iframe onto a file that is no longer served.

## Declaring in this file

```lua
Config.CustomApps = {
    exemple = {
        identifier = 'exemple',
        name = 'Exemple',
        description = "Une app d'exemple, sans interface.",
        developer = 'NASH',
        defaultApp = true,
        size = 128,
        onUse = function()
            exports['nash_phone']:SendNotification({
                app = 'exemple',
                title = 'Exemple',
                content = "L'app a bien été ouverte.",
            })
        end,
        onServerUse = function(source)
            print(('[nash_phone] joueur %s a ouvert l app d exemple'):format(source))
        end,
    },
}
```

The shipped file contains this example, commented out. It has no interface: it only fires a
notification, which is enough to check that everything is wired before writing any HTML.

The table key doubles as a fallback identifier, so `Config.CustomApps.mon_app = { name = ... }`
works without an explicit `identifier`.

`Config.CustomApps` is read once, when the phone's client resource starts.

## Fields

| Field | Description |
| ----- | ----------- |
| `identifier` | **Required.** Unique identifier, never shown to the player. It is the key: two apps cannot share one |
| `name` | Name displayed under the icon. Falls back to `identifier` |
| `description` | Description shown in the App Store |
| `developer` | Author name shown in the App Store |
| `icon` | Icon URL. For an image in **your** resource: `'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png'` |
| `ui` | Path to your interface's HTML, **prefixed with your resource name**: `'my-resource/ui/index.html'`. Without it, the app only calls `onUse` when tapped and opens nothing |
| `defaultApp` | `true` (default) = placed on the home screen from the first boot. `false` = has to be downloaded from the App Store |
| `game` | `true` = filed under the App Store's Games category |
| `size` | Weight announced in the App Store, in KB. Purely cosmetic |
| `images` | List of screenshot URLs, shown on the App Store page |
| `price` | In-game price to download it. `0` = free |
| `landscape` | `true` = the app displays in landscape |
| `keepOpen` | `true` = the phone does **not** close when the player opens the app. Reserve this for apps that take over the screen, a camera for instance |
| `onUse` | Function called **client-side** when the app is opened |
| `onServerUse` | Function called **server-side** when the app is opened. Receives the player's `source` |

{% hint style="info" %}
`price` is accepted for LB Phone compatibility, but the App Store never charges anything: every
app is displayed as free. Treat the field as inert.
{% endhint %}

## The three traps that cost the most time

**The `ui` path starts with the name of YOUR resource.** Writing `ui/index.html` would point the
iframe at the phone itself: a blank page, with no error. The phone refuses a path with no
resource name and tells you in the console:

```
[nash_phone] app personnalisée refusée (ma-ressource) : ui = 'ui/index.html' n'a pas de nom de ressource ; attendu 'ma-ressource/ui/index.html'
```

**Declare your files in `files{}` of your `fxmanifest.lua`.** An undeclared file is not served
to the player. Same symptom: a blank page, no message.

**Do not load your data from `onUse`.** The app opens before your interface has finished
loading, so the message would arrive in the void. Your interface is what asks, with `fetchNui`,
once it is ready.

## onServerUse only works from this file

A Lua function does not travel across the network. An app registered at runtime with
`exports['nash_phone']:AddCustomApp{...}` from a **client** script cannot hand its
`onServerUse` to the server: the function only exists in the client context.

Two options for those apps:

- Declare the app in `config/custom_apps.lua`, a shared file that both contexts see, or
- Listen for the public event from your own server script:

```lua
AddEventHandler('nash_phone:customAppUsed', function(src, identifier)
    if identifier ~= 'ma_banque' then return end
    -- ...
end)
```

That event fires on every custom app opening, whichever way the app was declared.

## Exports

| Export | Side | Description |
| ------ | ---- | ----------- |
| `exports['nash_phone']:AddCustomApp(app)` | Client | Registers or replaces an app. Returns `false` and prints the reason if the declaration is refused |
| `exports['nash_phone']:RemoveCustomApp(identifier)` | Client | Removes an app. Returns `false` if it was not registered |
| `exports['nash_phone']:SendCustomAppMessage(identifier, data)` | Client | Pushes a message to an app's interface. It comes out as `onNuiEvent(action, data)` |
| `exports['nash_phone']:SendCustomAppMessage(src, identifier, data)` | Server | Same thing from the server, for authoritative data such as a wallet total or an inventory |

## Config.CustomAppsSettings

```lua
Config.CustomAppsSettings = {
    max = 40,
    debug = false,
}
```

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `max` | `40` | Most custom apps accepted at once. A guard rail: a buggy resource calling `AddCustomApp` in a loop cannot drown the home screen. Replacing an existing app does not count against it |
| `debug` | `false` | Print every registration and removal to the console. Useful when an app does not appear and you want to know whether it was even declared |

When the cap is hit, the console says so and names the setting:

```
[nash_phone] app personnalisée « ma_banque » refusée : plafond de 40 atteint (Config.CustomAppsSettings.max)
```

## The interface bridge

```html
<script src="https://cfx-nui-nash_phone/custom_app_bridge/nash-bridge.js"></script>
```

Load it **before** your own script. Loading it from `nash_phone` rather than copying it means
your app follows the phone's updates. If you would rather freeze it, copy
`custom_app_bridge/nash-bridge.js` into your resource: it depends on nothing.

The bridge puts on `globalThis` exactly what LB Phone's documentation describes, under the same
names: `resourceName`, `appName`, `settings`, `components`, `fetchNui`, `onNuiEvent`,
`onSettingsChange`, `createCall`.

It exists because LB serves your app's interface from the same origin as the phone and can
write into its `globalThis` directly. Here your interface is served by **your** resource, a
different origin that no outside code can write into. The bridge is therefore loaded inside
your page and talks to the phone by messages. Same result, same API.

Nothing in it can crash your app. What is not wired yet returns a **neutral value and writes a
console warning**, so you know which one you are missing.

| Function | State |
| -------- | ----- |
| `fetchNui(event, data, scriptName?)` | Available |
| `onNuiEvent(action, handler)` | Available |
| `onSettingsChange(handler)` | Available |
| `resourceName`, `appName`, `settings` | Available |

| Component | State |
| --------- | ----- |
| `setPopUp` | Available |
| `setContextMenu` | Available |
| `setContactSelector` | Available |
| `setColorPicker` | Available |
| `setEmojiPickerVisible` | Available |
| `setGallery` | Available |
| `setFullscreenImage` | Available |
| `setHomeIndicatorVisible` | Available |
| `saveToGallery` | Available |
| `createCall` | Available |
| `uploadMedia` | Not wired yet |
| `setShareComponent` | Not wired yet |
| `setGifPickerVisible` | Not wired yet |
| `createGameRender` | Not wired yet |
| `GameMap` | Not wired yet |

### Theme

The bridge sets `data-theme="dark"` or `"light"` on `<html>` and updates it when the player
switches mode. Hook your CSS to it: an app that stays dark on a phone in light mode is spotted
immediately.

### Where your content goes

The status bar and the Dynamic Island are painted **by the phone, on top of you**. Leave **62 px
of inset at the top** and **34 px at the bottom** for the home indicator, as the example app
does.

## Applications written for another phone

The field names and the function names above are deliberately LB Phone's: an app written for LB
Phone declares itself here without changing anything in its table or in its interface code.

The only obstacle is the resource name it calls in its Lua.

- **If you can open its file:** replace that name with `nash_phone`. One line.
- **If it is escrowed:** use the `nash_lbcompat` resource, shipped separately, which answers to
  the expected name and forwards the calls. Read its `README.md` first: it has to be renamed,
  and it must never run alongside the original resource.

{% hint style="warning" %}
An app is only as portable as the functions it uses. The table above covers the interface side.
For the Lua side, see the [Developer API](../developer-api/README.md): it lists the exports
that are really wired. The others return neutral values, so the app starts without crashing but
without data.
{% endhint %}
