# config/custom_apps.lua

The registry of third-party applications: yours, or anyone else's.

File: `config/custom_apps.lua`, loaded as a shared script.

There are two ways to add an app, and the choice depends on one thing only: whether it has an
interface.

| Way | When to use it |
| --- | -------------- |
| **Here**, in `Config.CustomApps` | The simplest. You write the app in this file and it is declared on every phone on the server. Required if you need `onServerUse` |
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
survives a `stop` and opens an iframe onto a file that is no longer served. Players keep the
app installed: when the resource starts again, its icon comes back in the same place.

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
| `icon` | Icon URL. For an image in **your** resource: `'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png'`. Without it, or when the link fails to load, the icon is a coloured tile with the first letter of `name` |
| `ui` | Path to your interface's HTML, **prefixed with your resource name**: `'my-resource/ui/index.html'`. Without it, tapping the app runs `onUse` and `onServerUse` and opens a short screen saying the app has no interface |
| `defaultApp` | `true` (default) = installed and placed on the home screen (last page) **once per character**, including characters who already have a phone. `false` = the player gets it from the App Store. See below |
| `game` | `true` = listed under "Server Games" in the App Store and filed under Games in the App Library. Otherwise "Server Apps" and Utilities |
| `description` | Text of the app's App Store page, also shown under its name in App Store lists |
| `developer` | Publisher shown in the App Store. Defaults to "This server" |
| `size` | Size in **KB**, shown in the App Store. Without it, no size is shown |
| `images` | Image URLs, shown as screenshots on the App Store page |
| `price` | Nothing is ever charged. `0` or absent shows Free on the App Store page, above `0` the price row is hidden |
| `landscape` | `true` = the interface is displayed rotated, in landscape |
| `onUse` | Function called **client-side** when the app is opened |
| `onServerUse` | Function called **server-side** when the app is opened. Receives the player's `source` |
| `keepOpen` | Accepted for LB Phone compatibility, so an LB app's table works unchanged, but not used: opening an app never closes the phone anyway |

## App Store, App Library and home screen

Custom apps are listed in the App Store and behave like the phone's own App Store apps. Each
character's choices are saved on the server, in `nash_phone_home`.

| `defaultApp` | What the player gets |
| ------------ | -------------------- |
| `true` (or absent) | Installed and placed on the home screen the first time the app is declared while that character is loaded. **Only once:** an icon the player removed, or an app they deleted, does not come back on its own, after a reconnect or a server restart |
| `false` | Nothing until the player taps Get on the app's App Store page |

- **App Store:** under "Server Apps" on the Today tab, and under "Server Apps" or "Server Games"
  on the Apps and Games tabs. Get, Open and Delete work as for the phone's own apps.
- **App Library:** lists the app once it is installed, even when it is off the home screen. A
  long press on its icon offers "Add to Home Screen" (when the icon is off the home screen) and
  "Delete App".
- **Home screen:** the minus button offers "Remove from Home Screen" (the app stays installed and
  in the App Library) or "Delete App" (uninstalled until downloaded again).
- Switching an app from `false` to `true` later gives it, once, to the characters who never
  installed or deleted it.
- An app whose identifier is already used by a built-in or App Store app (`notes`, `snapz`...) is
  left out of the App Store and never installed automatically.

An app declared here is declared on every client, so every player sees it in the App Store. To
offer an app to some players only, declare it from your own resource, only on their clients:
see [Custom Applications](../developer-api/custom-apps.md#installation-and-the-app-store).

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
Phone usually declares itself here without changing anything in its table or in its interface
code.

The first obstacle is the resource name it calls in its Lua.

- **If you can open its file:** replace `exports['lb-phone']` with `exports['nash_phone']`, and
  `'lb-phone'` wherever the app waits for the phone (`GetResourceState`, `dependency` in its
  `fxmanifest.lua`).
- **If it is escrowed:** use the `nash_lbcompat` resource, shipped separately, which answers to
  the expected name and forwards the calls. Read its `README.md` first: it has to be renamed,
  and it must never run alongside the original resource.

An LB app declared with `defaultApp = false` is listed in the App Store, where players get it as
they would on LB Phone.

{% hint style="warning" %}
**Not a magic fix.** Compatibility depends on what each app uses. Most simple LB Phone apps work as they are: declaration, icon, live messages, calls to their own resource, notifications, phone number. An app will only partly work, or not at all, if it:

- relies on the functions LB Phone injects into its page (`fetchNui`, `resourceName`, `getSettings`, `onSettingsChange`, or components such as pop-ups, context menus, photo and contact pickers, the camera) without a fallback of its own;
- reads LB Phone's own database tables (for example `phone_phones` or `phone_messages`): NASH Phone stores its data in its own tables, so those queries find nothing;
- calls LB Phone functions NASH Phone has no equivalent for: they return a neutral value, so the matching feature stays empty;
- has a layout made for LB Phone's screen: NASH Phone's status bar and Dynamic Island cover the top 62 px.

Test every third-party app on a test server before promising it to your players.
{% endhint %}

{% hint style="info" %}
The table above covers the interface side.
For the Lua side, see the [Developer API](../developer-api/README.md): it lists the exports
that are really wired. The others return neutral values, so the app starts without crashing but
without data.
{% endhint %}
