# config/custom_apps.lua

The registry of third-party applications: yours, or anyone else's.

File: `config/custom_apps.lua`, loaded as a shared script.

There are two ways to add an app, and the choice depends on one thing only: whether it has an
interface.

| Way | When to use it |
| --- | -------------- |
| **Here**, in `Config.CustomApps` | The simplest. You write the app in this file and it appears on every phone on the server. Required if you need `onServerUse` |
| **From your own resource**, with the `AddCustomApp` export | Required if your app has an interface, because its HTML has to live in **your** resource |

A working reference resource, `nash_exemple_app`, ships alongside the phone. It is three Lua
files and one HTML page, and it exercises everything described below.

## Declaring from your own resource

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

That is the whole setup. Your interface now has `fetchNui`, `onNuiEvent`, `onSettingsChange`,
`createCall` and `components`.

Call `AddCustomApp` on the **client**, once the player is loaded. Apps registered by a
third-party resource are removed automatically when that resource stops, so an icon never
survives a `stop` and opens an iframe onto a file that is no longer served.

## Declaring in this file

```lua
Config.CustomApps = {
    example = {
        identifier = 'example',
        name = 'Example',
        description = 'An example app, without an interface.',
        developer = 'Nash Store',
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

The shipped file contains this example, commented out. It has no interface: it only fires a
notification, which is enough to check that everything is wired before writing any HTML.

The table key doubles as a fallback identifier, so `Config.CustomApps.my_app = { name = ... }`
works without an explicit `identifier` (`id` is also accepted in place of `identifier`).

`Config.CustomApps` is read once, when the phone's client resource starts.

## Fields

| Field | Description |
| ----- | ----------- |
| `identifier` | **Required.** Unique identifier, never shown to the player. It is the key: two apps cannot share one, and declaring the same identifier again replaces the app |
| `name` | Name displayed under the icon. Falls back to `identifier` |
| `icon` | Icon URL. For an image in **your** resource: `'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png'`. Without it, the icon is a coloured tile with the first letter of `name` |
| `ui` | Path to your interface's HTML, **prefixed with your resource name**: `'my-resource/ui/index.html'`. Without it, tapping the app runs `onUse` and `onServerUse` and opens a short screen saying the app has no interface |
| `defaultApp` | `true` (default) = placed on every player's home screen, including players who already have a phone (it is added on the last page). `false` = see the warning below |
| `landscape` | `true` = the interface is displayed rotated, in landscape |
| `onUse` | Function called **client-side** when the app is opened |
| `onServerUse` | Function called **server-side** when the app is opened. Receives the player's `source` |
| `description`, `developer`, `game`, `size`, `images`, `price`, `keepOpen` | Accepted for LB Phone compatibility, so an LB app's table works unchanged, but not used: custom apps have no App Store page, nothing is ever charged, and opening an app never closes the phone anyway |

{% hint style="warning" %}
**Custom apps are not listed in the App Store.** With `defaultApp = false`, the app is not placed
on the home screen, and it is removed from the home screens where it already was (unless the
player had installed it). Since it cannot be downloaded either, players have **no way to reach
it**. Keep `defaultApp = true` (or leave it out) for any app players should use.
{% endhint %}

## The three traps that cost the most time

**The `ui` path starts with the name of YOUR resource.** The first part of the path is read as
a resource name: `ui/index.html` would look for a resource called `ui`, and show a blank page
with no error. The phone only catches a path with no `/` at all, refuses the app and tells you
in the client console:

```
[nash_phone] app personnalisée refusée (my-resource) : ui = 'index.html' n'a pas de nom de ressource ; attendu 'my-resource/index.html'
```

Building the path with `GetCurrentResourceName() .. '/ui/index.html'`, as in the example above,
avoids the problem.

**Declare your files in `files{}` of your `fxmanifest.lua`.** An undeclared file is not served
to the player. Same symptom: a blank page, no message.

**Do not load your data from `onUse`.** The app opens before your interface has finished
loading, so the message would arrive in the void. Your interface is what asks, with `fetchNui`,
once it is ready.

## onServerUse only works from this file

A Lua function does not travel across the network. An app registered at runtime with
`exports['nash_phone']:AddCustomApp{...}` from a **client** script cannot hand its
`onServerUse` to the server: the function only exists in the client context.

The phone also fires a public server event when a custom app is opened:

```lua
AddEventHandler('nash_phone:customAppUsed', function(src, identifier)
    if identifier ~= 'my_bank' then return end
    -- ...
end)
```

It goes through the same path as `onServerUse`: it fires only for an app whose declaration has
an `onServerUse` function, in practice an app declared in `config/custom_apps.lua` (an empty
`onServerUse = function() end` is enough to get the event).

For an app registered from your own resource, let your interface say it was opened: it already
talks to your resource, since `fetchNui(event, data)` posts to the resource that registered the
app. Your client script receives that call and triggers your own server event.

## Exports

| Export | Side | Description |
| ------ | ---- | ----------- |
| `exports['nash_phone']:AddCustomApp(app)` | Client | Registers or replaces an app. Returns `false` and prints the reason if the declaration is refused |
| `exports['nash_phone']:RemoveCustomApp(identifier)` | Client | Removes an app. Returns `false` if it was not registered |
| `exports['nash_phone']:SendCustomAppMessage(identifier, data)` | Client | Pushes a message to an app's interface. It comes out as `onNuiEvent(action, data)` |
| `exports['nash_phone']:SendCustomAppMessage(src, identifier, data)` | Server | Same thing from the server, for authoritative data such as a wallet total or an inventory |
| `exports['nash_phone']:GetCustomApps()` | Client | Returns the list of registered custom apps, sorted by identifier |

## Config.CustomAppsSettings

```lua
Config.CustomAppsSettings = {
    max = 40,
    debug = false,
}
```

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `max` | `40` | Most custom apps accepted at once. A guard rail: a buggy resource calling `AddCustomApp` in a loop cannot drown the home screen. Replacing an existing app does not count against it. A value that is not a positive number gives `40` |
| `debug` | `false` | Print every registration and removal to the client console (F8). Useful when an app does not appear and you want to know whether it was even declared |

When the cap is hit, the console says so and names the setting:

```
[nash_phone] app personnalisée « my_bank » refusée : plafond de 40 atteint (Config.CustomAppsSettings.max)
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
