# Custom Applications

Any FiveM resource can put its own app on the phone's home screen. The app ships with your resource, its interface is your own HTML, and NASH Phone mounts it inside the chassis.

A complete working model is delivered next to the phone as the `nash_exemple_app` resource: three Lua files and one HTML page, exercising everything on this page.

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

```lua
-- fxmanifest.lua of YOUR resource
files {
    'ui/index.html',
    'ui/**',
}
```

Your page now has `fetchNui`, `onNuiEvent`, `onSettingsChange`, `createCall` and `components`.

{% hint style="warning" %}
The three mistakes that cost the most time, all of which produce a **blank page with no error**:

1. **`ui` must start with your resource name.** `'ui/index.html'` points the iframe at the phone itself. The phone rejects a path with no `/` and says so in the console, but `'nash_phone/whatever'` would be accepted and silently wrong.
2. **Every file must be declared in `files{}`.** An undeclared file is not served to the player.
3. **Do not push your data from `onUse`.** The app opens before your interface has finished loading; the message lands in the void. Your interface asks for data with `fetchNui` once it is mounted.
{% endhint %}

## Declaring an app

Two entry points, one registry.

<details>

<summary>From your own resource: AddCustomApp (client)</summary>

The normal path, and the only one that works when your app has an interface, because the HTML must live in your resource.

```lua
local ok = exports['nash_phone']:AddCustomApp(app)
```

**Returns:** `boolean`. `false` when the declaration was refused; the reason is printed to the console.

Call it **client-side**, after the phone resource is started. `GetInvokingResource()` records which resource declared the app, which is what routes `fetchNui` later and what allows the automatic cleanup below.

Calling `AddCustomApp` again with the same `identifier` replaces the previous declaration.

</details>

<details>

<summary>From the phone's config: Config.CustomApps</summary>

`config/custom_apps.lua` is a shared file, so both contexts see the table. This is the only way to use `onServerUse`, since a Lua function cannot cross the network.

```lua
Config.CustomApps = {
    example = {
        identifier = 'example',
        name = 'Example',
        description = 'An example app, no interface.',
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
            print(('player %s opened the example app'):format(source))
        end,
    },
}
```

The table key is used as a fallback identifier, so `Config.CustomApps.my_app = { name = '…' }` works without an explicit `identifier`.

Apps declared here are registered on resource start, which is often **before** the interface has mounted. The page asks for the list again when it is ready, so nothing is lost.

</details>

<details>

<summary>RemoveCustomApp / GetCustomApps</summary>

```lua
exports['nash_phone']:RemoveCustomApp('my_bank')   -- boolean, false if unknown
local apps = exports['nash_phone']:GetCustomApps() -- read-only list
```

You rarely need `RemoveCustomApp`: when your resource stops, the phone removes every app it declared and refreshes the home screen. Without that, the icon would stay and opening it would load an iframe pointing at a file that is no longer served: a blank page with no error.

</details>

### Declaration fields

| Field | Type | Description |
|---|---|---|
| `identifier` | `string` | **Required.** Unique key, never shown to the player. Two apps cannot share one |
| `name` | `string` | Label under the icon. Defaults to `identifier` |
| `description` | `string` | Shown on the App Store page |
| `developer` | `string` | Author, shown on the App Store page |
| `icon` | `string` | Icon URL. For an image in your resource: `'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png'` |
| `ui` | `string` | Path to your HTML, **prefixed with your resource name**. Without it the app only calls `onUse` and opens nothing |
| `defaultApp` | `boolean` | `true` (default) puts it on the home screen from the first boot. `false` means it has to be downloaded from the App Store |
| `game` | `boolean` | Files it under the App Store's Games category |
| `size` | `number` | Weight announced in the App Store, in KB. Cosmetic only |
| `images` | `table` | Array of screenshot URLs for the App Store page |
| `price` | `number` | In-game price to download it. `0` is free |
| `landscape` | `boolean` | Renders the app in landscape (the chassis stays upright, the interface rotates) |
| `keepOpen` | `boolean` | The phone does **not** close when the player opens the app. For apps that take over the screen, such as a camera |
| `onUse` | `function` | Called **client-side** when the app is opened |
| `onServerUse` | `function` | Called **server-side** with the player's `source`. Only works from `Config.CustomApps`: see below |

`size` and `price` are coerced with `tonumber` and fall back to `0`.

{% hint style="info" %}
`Config.CustomAppsSettings.max` caps how many custom apps can be registered at once, default `40`. A buggy resource calling `AddCustomApp` in a loop cannot flood the home screen. Set `Config.CustomAppsSettings.debug = true` to print every add and remove: the fastest way to tell "not declared" from "declared but not displayed".
{% endhint %}

## The open hooks

Opening an app runs, in order: `onUse` on the client, then: if the declaration carries one: `onServerUse` on the server, then the public event.

<details>

<summary>onUse: client</summary>

Runs in the client context of the player who opened the app. It is wrapped in `pcall`: an error in your code is printed but does not break the NUI callback, so the app never hangs on its launch screen.

```lua
onUse = function()
    print("the player just opened the app")
end,
```

</details>

<details>

<summary>onServerUse: server, config only</summary>

Receives the player's `source`. Also wrapped in `pcall`, with the failure printed as `onServerUse de « … » a échoué`.

{% hint style="warning" %}
`onServerUse` **only works for apps declared in `Config.CustomApps`.** An app declared at runtime by a client script cannot deliver this function to the server: a Lua function does not cross the network, and the client-side `nash_phone:customApp:use` event is deliberately restricted to names present in the phone's own config, since it comes from an untrusted source. For a runtime-declared app, listen to the public event instead.
{% endhint %}

</details>

<details>

<summary>nash_phone:customAppUsed: server event</summary>

The supported server-side entry point for apps declared at runtime.

```lua
-- server.lua of YOUR resource
AddEventHandler('nash_phone:customAppUsed', function(src, identifier)
    if identifier ~= 'my_bank' then return end
    print(('player %s opened the app'):format(src))
end)
```

It fires for **every** custom app on the server, so always filter on `identifier`. Note the underscore prefix (`nash_phone:`, not `nash-phone:`). Full description in [Server Events](server-events.md).

</details>

## The iframe bridge

Your page is served by **your** resource, not by the phone, so it is a different origin: the phone cannot write into its `globalThis`. The bridge is a small script loaded **inside your page** that talks to the phone by `postMessage`, and it recreates the same API surface.

```html
<script src="https://cfx-nui-nash_phone/custom_app_bridge/nash-bridge.js"></script>
```

Loading it from `nash_phone` keeps your app in step with phone updates. The file depends on nothing, so you can also copy it into your own resource to pin it.

### What it puts on `globalThis`

| Name | Type | Description |
|---|---|---|
| `resourceName` | `string` | The resource that declared the app. Default target of `fetchNui` |
| `appName` | `string` | Display name of the app |
| `settings` | `object` | The phone's settings, refreshed on every change |
| `fetchNui(event, data, scriptName?)` | `function` | Calls a `RegisterNUICallback` and returns its answer as a Promise |
| `onNuiEvent(action, handler)` | `function` | Listens to pushes from Lua |
| `onSettingsChange(handler)` | `function` | Fires when the phone's settings change |
| `createCall(options)` | `function` | Starts a call from the app |
| `components` | `object` | System sheets painted by the phone: see below |

### fetchNui

```js
const infos = await fetchNui('getInfos', { page: 1 })
```

No routing is involved: your page is served by your resource, so a `fetch` to your own origin lands naturally on your own `RegisterNUICallback`. `scriptName` is only there to target **another** resource.

```lua
-- client.lua of YOUR resource
RegisterNUICallback('getInfos', function(data, cb)
    cb({ name = GetPlayerName(PlayerId()) })
end)
```

### Pushing from Lua to the interface

Two exports, same effect, different context. Use the server one when the data is authoritative (a balance, an inventory) and you do not want the client computing it.

```lua
-- client
exports['nash_phone']:SendCustomAppMessage('my_bank', { action = 'tick', data = { hour = 12 } })

-- server
exports['nash_phone']:SendCustomAppMessage(src, 'my_bank', { action = 'tick', data = { hour = 12 } })
```

```js
onNuiEvent('tick', (data) => {
  console.log(data.hour)
})
```

{% hint style="warning" %}
The table you pass **must** be shaped `{ action = '…', data = { … } }`: the same shape as `SendNUIMessage`. `onNuiEvent` dispatches on `action` and hands your handler the `data` field only. A flat table is delivered but matches no listener.
{% endhint %}

Both return `boolean`. The client export returns `false` when the identifier is not registered; the server export returns `false` on a malformed `src` or `identifier`. The message is only relayed to the app whose identifier matches, so two custom apps open at once do not see each other's traffic.

### Theme

The bridge sets `data-theme="dark"` or `"light"` on `<html>` and updates it when the player switches mode. Hook your CSS to it: an app that stays dark on a light phone is spotted immediately.

```css
[data-theme='light'] body { background: #fff; color: #000; }
[data-theme='dark']  body { background: #000; color: #fff; }
```

The attribute is `light` only when `settings.darkMode === false`; anything else, including a context that has not arrived yet, renders as `dark`.

### Where your content can go

The status bar and the Dynamic Island are painted **by the phone, on top of your iframe**. Leave **62 px of padding at the top** and **34 px at the bottom** for the home indicator, as the example app does.

```css
body { padding: 62px 20px 34px; }
```

In landscape (`landscape = true`) the frame is rotated to 874 × 402 px; the chassis itself stays upright in the player's hand.

## Components

Sheets, pickers and full-screen overlays must cover the whole display, status bar included, or they immediately read as "not the system". Your iframe cannot draw outside itself, so the phone paints them and your app only asks.

Every call returns a Promise. It resolves **when the sheet closes**, not when it opens. Only one component is open at a time, as on iOS.

| Component | State | Call |
|---|---|---|
| `setPopUp` | available | `components.setPopUp({ title, description, buttons, attachment })` |
| `setContextMenu` | available | `components.setContextMenu({ title, buttons })` |
| `setContactSelector` | available | `components.setContactSelector({ onSelect })` |
| `setColorPicker` | available | `components.setColorPicker({ onSelect, onClose })` |
| `setEmojiPickerVisible` | available | `components.setEmojiPickerVisible({ onSelect })` |
| `setGallery` | available | `components.setGallery({ onSelect, includeImages, includeVideos, multiSelect })` |
| `setFullscreenImage` | available | `components.setFullscreenImage(url)` |
| `setHomeIndicatorVisible` | available | `components.setHomeIndicatorVisible(true \| false)` |
| `saveToGallery` | available | `components.saveToGallery(url)`: resolves to the photo id, or `null` |
| `createCall` | available | `createCall({ number, videoCall, hideNumber })` |
| `uploadMedia` | not wired | resolves `null` |
| `setShareComponent` | not wired | resolves `null` |
| `setGifPickerVisible` | not wired | resolves `null` |
| `createGameRender` | not wired | resolves a handle whose calls do nothing |
| `GameMap` | not wired | resolves `null` on every method |

{% hint style="info" %}
Nothing on this list can crash your app. A component that is not wired resolves to a **neutral value and prints a console warning naming it**, so you know which one you are missing instead of debugging a silent Promise.
{% endhint %}

**Buttons:** `buttons` is an array of `{ title, color, cb }`. `cb` is a function; see the callback note below.

```js
await components.setPopUp({
  title: 'Confirm',
  description: 'Transfer 500 $ ?',
  buttons: [
    { title: 'Cancel' },
    { title: 'Confirm', color: '#34c759', cb: () => doTransfer() },
  ],
})
```

**Emoji picker:** pass an object. `setEmojiPickerVisible({ onSelect })` opens it, `setEmojiPickerVisible({ visible: false })` closes it. An empty object or a bare boolean is read as a close, not an open.

**Colour picker:** the palette is the iOS system set plus black and white, 16 swatches. There is no free-form colour input.

**Gallery:** `includeImages` and `includeVideos` default to `true`, `multiSelect` to `false`.

**saveToGallery:** goes through the same route as the Lua `SaveToGallery` export. There is no separate "add to gallery" path.

{% hint style="warning" %}
**Functions do not cross `postMessage`.** The bridge stores each callback under a token (`{ __nashFn: 'fnN' }`), sends only the token, and replays your function when the player acts. This is transparent as long as your callbacks are declared in the object you pass. It also means a callback stays alive in your page: do not rely on it being garbage-collected when the sheet closes.
{% endhint %}

## The message protocol

You do not need this if you use `nash-bridge.js`. It is here for anyone writing their own bridge or debugging one.

**App → phone** (`postMessage` to `parent`)

| Message | Meaning |
|---|---|
| `{ source: 'nash-phone-app', type: 'ready' }` | The app's bridge is loaded. The phone answers with `context` |
| `{ source: 'nash-phone-app', type: 'component', id, name, args }` | Requests a component. `id` is echoed back in `component-result` |

**Phone → app**

| Message | Meaning |
|---|---|
| `{ source: 'nash-phone', type: 'context', payload }` | `payload` = `{ resourceName, appName, appIdentifier, settings }` |
| `{ source: 'nash-phone', type: 'settings', payload }` | The phone's settings changed |
| `{ source: 'nash-phone', type: 'nui-event', payload }` | Relay of `SendCustomAppMessage`. `payload` = `{ action, data }` |
| `{ source: 'nash-phone', type: 'component-result', id, result }` | The component identified by `id` closed |
| `{ source: 'nash-phone', type: 'invoke', fn, args }` | Replay one of your callbacks, by token |

Nothing is pushed before the `ready` handshake: a message sent before your app listens would be lost.

The iframe runs with `sandbox="allow-scripts allow-same-origin allow-forms allow-popups"`. `allow-same-origin` is required: without it your page could not call its own NUI callbacks, which are requests to its own origin.

## Apps written for another phone

Field and function names are deliberately those of LB Phone: an app written for LB Phone declares itself here without a single change to its table or to its interface code.

The only obstacle is the resource name hardcoded in its Lua.

- **If you can open its files:** replace that string with `nash_phone`. One line.
- **If it is escrowed:** use the `nash_lbcompat` resource, delivered separately. It answers to the expected name and forwards the calls. Read its `README.md` first: it has to be renamed, and it must never run alongside the original resource.

An app is only as portable as the exports it uses. Those actually wired are listed in [Server Exports](server-exports.md) and [Client Exports](client-exports.md); the rest return neutral values, so the app launches without crashing but without data.

## Debugging checklist

| Symptom | Check |
|---|---|
| Icon never appears | Set `Config.CustomAppsSettings.debug = true`. No "ajout" line means the declaration was refused: the reason is on the console line above |
| Icon appears, page is blank | `ui` does not start with your resource name, or the file is missing from `files{}` in your `fxmanifest.lua` |
| Blank in the browser demo build | Expected. `nui://` does not exist in a browser; the phone shows an explanatory message instead of an empty frame |
| `fetchNui` rejects with "contexte pas encore reçu" | You called it before the `ready` handshake completed. Call it from your mount code, not at script top level |
| A component resolves instantly with `null` | It is not wired yet. Its name is in the console warning |
| Icon stays after you stop your resource | Should not happen: the phone removes it on `onClientResourceStop`. If it persists, the app was declared by `nash_phone` itself through `Config.CustomApps` |

## Related

- [Server Events](server-events.md): `nash_phone:customAppUsed`
- [Client Events](client-events.md): `nash-phone:phoneToggled`, state bags
- [Commands](commands.md)
