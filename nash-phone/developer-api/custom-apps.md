# Custom Applications

Any FiveM resource can put its own app on the phone: on the home screen, in the App Store and in the App Library. The app ships with your resource, its interface is your own HTML, and NASH Phone mounts it inside the chassis.

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

1. **`ui` must start with your resource name.** `'ui/index.html'` points the iframe at a resource called `ui`. The phone rejects a path with no `/` and says so in the console, but `'nash_phone/whatever'` would be accepted and silently wrong.
2. **Every file must be declared in `files{}`.** An undeclared file is not served to the player.
3. **Do not push your data from `onUse`.** The app opens before your interface has finished loading; the message lands in the void. Your interface asks for data with `fetchNui` once it is mounted.
{% endhint %}

## Declaring an app

Two entry points, one registry.

<details>

<summary>From your own resource: AddCustomApp (client)</summary>

The normal path, and the only one that works when your app has an interface, because the HTML must live in your resource.

```lua
local ok, reason = exports['nash_phone']:AddCustomApp(app)
```

**Returns:** `true`, or `false` and a `reason` string when the declaration was refused. The same reason is printed to the client console (F8), even with `Config.CustomAppsSettings.debug` off. It is refused when the argument is not a table, when `identifier` is missing, when `ui` has no `/`, or when the cap of `Config.CustomAppsSettings.max` apps is reached.

Call it **client-side**, after the phone resource is started. The phone records which resource declared the app: that is what routes `fetchNui` later and what allows the automatic cleanup below. By default it is the calling resource (`GetInvokingResource()`). An optional second argument names another resource instead, and is used only if that resource is started or starting. It exists for forwarding resources such as the LB Phone compat; your own resource never needs it.

Calling `AddCustomApp` again with the same `identifier` replaces the previous declaration.

The phone keeps declared apps in memory only. If `nash_phone` restarts, your app is no longer registered until you declare it again, so declare it on the phone's start too:

```lua
AddEventHandler('onClientResourceStart', function(res)
    if res == 'nash_phone' then declareMyApp() end
end)
```

</details>

<details>

<summary>From the phone's config: Config.CustomApps</summary>

`config/custom_apps.lua` is a shared file, so both contexts see the table. This is the only way to use `onServerUse`, since a Lua function cannot cross the network.

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

Removing an app, either way, only hides it. Its icon, its App Store listing and its App Library entry disappear, but each character keeps it installed and its icon keeps its place on the home screen. When the app is declared again, for example when your resource restarts, it comes back where it was.

</details>

### Declaration fields

| Field | Type | Description |
|---|---|---|
| `identifier` | `string` | **Required.** Unique key, never shown to the player. Declaring an app with an identifier already in use replaces the previous one. `id` is accepted as an alias |
| `name` | `string` | Label under the icon. Defaults to `identifier` |
| `icon` | `string` | Icon URL. For an image in your resource: `'https://cfx-nui-' .. GetCurrentResourceName() .. '/ui/icon.png'`. Without it, or when the link fails to load, the icon is a coloured tile with the first letter of `name` |
| `ui` | `string` | Path to your HTML, **prefixed with your resource name**: the phone loads `nui://<ui>`. A value that already starts with `https://`, `http://` or `nui://` is used as is. Without `ui` the app only calls `onUse` and shows a short "no interface" message |
| `defaultApp` | `boolean` | `true` (default): installed and placed on the home screen **once per character**. `false`: the player gets it from the App Store. See [Installation and the App Store](#installation-and-the-app-store) |
| `game` | `boolean` | `true` lists the app under "Server Games" in the App Store and files it under Games in the App Library. Otherwise it goes under "Server Apps" and Utilities |
| `description` | `string` | Text of the app's App Store page, also shown under its name in App Store lists. A default text replaces it when missing |
| `developer` | `string` | Publisher shown in the App Store. Defaults to "This server" |
| `size` | `number` | Size in **KB**, shown in the App Store and counted in Settings > Storage. Without it, the App Store shows no size and Storage counts the app at 120 MB |
| `images` | `string[]` | Image URLs, shown as screenshots on the app's App Store page |
| `price` | `number` | Nothing is ever charged. `0` or absent: the App Store page reads Free. Above `0`: the price row is hidden |
| `landscape` | `boolean` | Renders the app in landscape (the chassis stays upright, the interface rotates) |
| `onUse` | `function` | Called **client-side** when the app is opened |
| `onServerUse` | `function` | Called **server-side** with the player's `source`. Only works from `Config.CustomApps`: see below |
| `keepOpen` | `boolean` | Accepted so a table written for LB Phone loads unchanged, **no effect** in NASH Phone |
| Any other field | | Ignored without error, for example `fixBlur` |

### Installation and the App Store

Custom apps behave like the phone's own App Store apps, and each character's choices are saved on the server, in `nash_phone_home`.

| `defaultApp` | What the player gets |
|---|---|
| `true` or absent | The app is installed and its icon placed on the last home page (or on a new page when that one is full) the first time it is declared while that character is loaded. **Only once:** if the player removes the icon or deletes the app, it does not come back on its own, after reopening the phone, reconnecting or a server restart |
| `false` | Nothing until the player taps Get on the app's App Store page. It is then installed and placed like a store app, and stays installed |

- **App Store.** Every declared app is listed under "Server Apps" on the Today tab, and on the Apps tab ("Server Apps") or the Games tab ("Server Games") depending on `game`. It also shows in search and in the account page's installed apps. Its page shows the icon, name, developer, description, size and `images`, with no ratings. Get, Open and Delete work as for the phone's own apps. The publisher is the app's own `developer`: `Config.Brand.studio` (`config/main.lua`) only applies to the phone's own apps.
- **App Library.** An installed app is listed there, under Games or Utilities. A long press on its icon opens a menu with "Add to Home Screen" (only when the icon is off the home screen) and "Delete App".
- **Home screen.** The minus button opens the iOS alert. "Remove from Home Screen" keeps the app installed, in the App Library, Spotlight and the App Store. "Delete App" uninstalls it until the player downloads it again from the App Store.
- **Back on the home screen.** An app taken off the home screen is still opened from the App Library, Spotlight or the App Store. "Add to Home Screen" in the App Library puts its icon back, after the last item of the last page.
- **Changing `defaultApp` later.** Switching an app from `false` to `true` gives it, once, to the characters who never installed or deleted it.
- **Identifier clash.** An app whose identifier is already used by a built-in or App Store app (`notes`, `snapz`...) is left out of the App Store and never installed automatically.

{% hint style="info" %}
**Offering an app to some players only.** The App Store lists every app declared on the player's client. To keep one for a job or an item holder, declare it only on their clients, and call `RemoveCustomApp` when they lose access: the icon goes, and comes back in place if they get access again. The client export `SetAppInstalled(identifier, true)` installs it for a player without the App Store, `SetAppInstalled(identifier, false)` uninstalls it; both are saved as that character's choice. `OpenApp(identifier)` opens it directly. See [Client Exports](client-exports.md).
{% endhint %}

{% hint style="info" %}
`Config.CustomAppsSettings.max` caps how many custom apps can be registered at once, default `40`. A buggy resource calling `AddCustomApp` in a loop cannot flood the home screen. Set `Config.CustomAppsSettings.debug = true` to print every add and remove: the fastest way to tell "not declared" from "declared but not displayed".
{% endhint %}

## The open hooks

Opening an app runs, in order: `onUse` on the client, then, only if the declaration carries an `onServerUse` function, a message to the server, which calls that `onServerUse` (for an app of `Config.CustomApps`) and fires the public event `nash_phone:customAppUsed`. An app declared without `onServerUse` stops after `onUse`: nothing reaches the server.

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
`onServerUse` **only works for apps declared in `Config.CustomApps`.** An app declared at runtime by a client script cannot deliver this function to the server: a Lua function does not cross the network, and the server only looks for `onServerUse` in the phone's own config. For an app declared from your own resource, use your own NUI callback instead (see below).
{% endhint %}

</details>

<details>

<summary>nash_phone:customAppUsed: server event</summary>

Fired on the server right after `onServerUse`, so only for apps whose declaration carries an `onServerUse` function.

```lua
-- server.lua of YOUR resource
AddEventHandler('nash_phone:customAppUsed', function(src, identifier)
    if identifier ~= 'example' then return end
    print(('player %s opened the app'):format(src))
end)
```

Always filter on `identifier`, and do not grant anything on this event alone: the identifier comes from the player's client. Note the underscore prefix (`nash_phone:`, not `nash-phone:`). Full description in [Server Events](server-events.md).

</details>

<details>

<summary>Reacting on the server to an app declared from your resource</summary>

An app declared with `AddCustomApp` has no reliable server hook in the phone. Its server entry point is your own interface: once the page is mounted, it calls one of your NUI callbacks, which triggers your own server event.

```js
// ui/index.html of YOUR resource. The third argument names your resource explicitly,
// so the call works even before the phone has sent its context to the page.
fetchNui('opened', {}, 'my_resource')
```

```lua
-- client.lua of YOUR resource
RegisterNUICallback('opened', function(_, cb)
    TriggerServerEvent('my_bank:opened')
    cb({ ok = true })
end)
```

```lua
-- server.lua of YOUR resource
RegisterNetEvent('my_bank:opened', function()
    local src = source
    -- Your server-side reaction, with your own checks
end)
```

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

No routing is involved: your page is served by your resource, so a `fetch` to your own origin lands naturally on your own `RegisterNUICallback`. The Promise resolves to the parsed JSON answer, or `null` when the answer is not JSON.

Without `scriptName`, the target is `resourceName`, which only exists once the phone has sent its context to the page, shortly after the bridge loads. A call made before that rejects with `nash-bridge : contexte pas encore reçu`. Passing your own resource name as `scriptName` removes that wait; it can also target **another** resource.

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

The same table is also posted to your page **raw**, right after the bridge's envelope, so that `event.data` is your `{ action, data }` itself: this is the format apps written for LB Phone listen to (see [Apps written for LB Phone](#apps-written-for-lb-phone)). The bridge ignores the raw copy. A page with its own `message` listener receives both, and recognises the envelope by `event.data.source === 'nash-phone'`. Nothing is posted raw for a `nil` payload.

Both return `boolean`. The client export returns `false` when the identifier is not registered; the server export returns `false` on a malformed `src` or `identifier` (it does not check that the app exists). The message is only relayed to the app whose identifier matches, so two custom apps open at once do not see each other's traffic.

A message only reaches your page while the app is open on screen and its page has loaded the bridge. Nothing is queued: a message sent while the app is closed, or before the page has finished loading, is lost. This is why your page should ask for its data with `fetchNui` rather than wait for a push.

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

### Keyboard

As soon as the player clicks inside your page, the keyboard goes to your app only, exactly as in the phone's own text fields: the character stops moving, so typing never walks or closes the phone. Escape and Backspace stay inside your page. A tap outside the page (status bar, home indicator) or the home gesture gives the keys back to the game. This applies to every custom app, LB or not.

## Components

Sheets, pickers and full-screen overlays must cover the whole display, status bar included, or they immediately read as "not the system". Your iframe cannot draw outside itself, so the phone paints them and your app only asks.

Every call returns a Promise. For a sheet, it resolves **when the sheet closes**, not when it opens. Only one component is open at a time, as on iOS: opening a second one replaces the first, and the Promise of the first never resolves.

| Component | State | Call | Resolves to |
|---|---|---|---|
| `setPopUp` | available | `components.setPopUp({ title, description, buttons, attachment })` | Index of the button pressed (from `0`), or `null` when dismissed by tapping outside |
| `setContextMenu` | available | `components.setContextMenu({ title, buttons })` | Index of the button pressed, or `null` on Cancel |
| `setContactSelector` | available | `components.setContactSelector({ onSelect })` | `{ firstname, lastname, number, avatar, display }`, or `null` |
| `setColorPicker` | available | `components.setColorPicker({ onSelect, onClose })` | The colour picked, as `'#rrggbb'`, or `null` |
| `setEmojiPickerVisible` | available | `components.setEmojiPickerVisible({ onSelect })` | The emoji picked, or `null` |
| `setGallery` | available | `components.setGallery({ onSelect, includeImages, includeVideos, multiSelect })` | `{ src, isVideo, id }`, an array of them with `multiSelect`, or `null` |
| `setFullscreenImage` | available | `components.setFullscreenImage(url)` | `null`, when the player taps to close |
| `setHomeIndicatorVisible` | accepted, no effect | `components.setHomeIndicatorVisible(true \| false)` | `true`. The home indicator does not change today |
| `saveToGallery` | available | `components.saveToGallery(url)` | The photo id, or `null` when refused |
| `createCall` | available | `createCall({ number, videoCall, hideNumber })` | `{ ok: true, data: { id, … } }`, `{ ok: false, error }` (for example `offline`, `busy`), or `null` |
| `uploadMedia` | not wired | `components.uploadMedia(type, blob)` | `null` |
| `setShareComponent` | not wired | | `null` |
| `setGifPickerVisible` | not wired | | `null` |
| `createGameRender` | not wired | `components.createGameRender(canvas)` | A handle whose calls do nothing |
| `GameMap` | not wired | `components.GameMap(container, options)` | Returns an object at once; its `ready` and every method resolve `null` |

{% hint style="info" %}
Nothing on this list can crash your app. A component that is not wired resolves to a **neutral value and prints a console warning naming it**, so you know which one you are missing instead of debugging a silent Promise.
{% endhint %}

**Buttons:** `buttons` is an array of `{ title, color, cb }`. `cb` is a function; see the callback note below. `color` is a colour **name**, not a hex code: `'red'` or `'green'` in a pop-up, `'red'` in a context menu. Any other value, or none, gives the system blue.

```js
await components.setPopUp({
  title: 'Confirm',
  description: 'Transfer 500 $ ?',
  buttons: [
    { title: 'Cancel' },
    { title: 'Confirm', color: 'green', cb: () => doTransfer() },
  ],
})
```

**Pop-up attachment:** `attachment` is `{ src }`, an image shown above the title.

**Contact selector:** lists the player's contacts. `onSelect` receives the same object the Promise resolves to.

**Emoji picker:** pass an object. `setEmojiPickerVisible({ onSelect })` opens it, `setEmojiPickerVisible({ visible: false })` closes it. An empty object or a bare boolean is read as a close, not an open. `onSelect` receives `{ emoji }`. The picker offers a fixed set of 24 emoji.

**Colour picker:** the palette is the iOS system set plus black and white, 16 swatches. There is no free-form colour input. `onSelect` receives the colour; `onClose` is accepted but never called, so read the `null` of the Promise to detect a dismissal.

**Gallery:** `includeImages` and `includeVideos` default to `true`, `multiSelect` to `false`. `onSelect` receives the same value the Promise resolves to.

**saveToGallery:** goes through the same route as the Lua `SaveToGallery` export, with the same rules (an `https://` address on a host of `Config.Gallery.Import.AllowedHosts`). There is no separate "add to gallery" path.

**createCall:** places a call from the player to `number`, audio or video (`videoCall: true`). `hideNumber` is accepted but ignored: whether the number is withheld follows the player's own "hidden number" setting.

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
| `{ source: 'nash-phone', type: 'nui-event', payload }` | Relay of `SendCustomAppMessage`. `payload` is the table you passed, normally `{ action, data }` |
| The `payload` itself, with no envelope | Same relay, posted right after the `nui-event` message, for apps written for LB Phone. Not sent when the payload is `nil`, or when it carries `source: 'nash-phone'` or `'nash-phone-app'` |
| `{ source: 'nash-phone', type: 'component-result', id, result }` | The component identified by `id` closed |
| `{ source: 'nash-phone', type: 'invoke', fn, args }` | Replay one of your callbacks, by token |

`context` and `settings` wait for the `ready` handshake. Relays of `SendCustomAppMessage`, in either form, do not: they are posted as soon as they arrive, so one sent before your page listens is lost.

The iframe runs with `sandbox="allow-scripts allow-same-origin allow-forms allow-popups"`. `allow-same-origin` is required: without it your page could not call its own NUI callbacks, which are requests to its own origin.

## Apps written for LB Phone

Most apps written for LB Phone run on NASH Phone without changes to their declaration table or their interface code. Field and function names are deliberately LB Phone's, and live messages are delivered in both formats.

{% hint style="warning" %}
**Not a magic fix.** Compatibility depends on what each app uses. Most simple LB Phone apps work as they are: declaration, icon, live messages, calls to their own resource, notifications, phone number. An app will only partly work, or not at all, if it:

- relies on the functions LB Phone injects into its page (`fetchNui`, `resourceName`, `getSettings`, `onSettingsChange`, or components such as pop-ups, context menus, photo and contact pickers, the camera) without a fallback of its own;
- reads LB Phone's own database tables (for example `phone_phones` or `phone_messages`): NASH Phone stores its data in its own tables, so those queries find nothing;
- calls LB Phone functions NASH Phone has no equivalent for: they return a neutral value, so the matching feature stays empty;
- has a layout made for LB Phone's screen: NASH Phone's status bar and Dynamic Island cover the top 62 px.

Test every third-party app on a test server before promising it to your players.
{% endhint %}

### Two ways to run one

| The app | What to do |
|---|---|
| **Editable**: you can open its Lua | Replace `exports['lb-phone']` with `exports['nash_phone']`. Replace `'lb-phone'` too wherever the app waits for the phone (`GetResourceState`, a resource name check in `onResourceStart`, `dependency` in its `fxmanifest.lua`), or it waits for a resource that never starts. No extra resource |
| **Escrowed**: its Lua cannot be opened | Use the `nash_lbcompat` resource, delivered separately, **renamed `lb-phone`** and started **after** `nash_phone`. Never run it next to the real `lb-phone`: two resources cannot share a name. Read its `README.md` first |

```cfg
ensure nash_phone
ensure lb-phone   # the nash_lbcompat folder, renamed
ensure my_lb_app
```

What the compat does:

- Forwards each call to `nash_phone`. The forwarded exports are listed at the top of its `client.lua` and `server.lua`; a missing one is added by name to the `RELAYES` table. `debug = true` in its `config.lua` prints every forwarded call.
- Records the app's own resource as the owner of each app declared through it, so the icon disappears when that resource stops.
- Remembers those apps and declares them again whenever `nash_phone` starts or restarts. An app declared while `nash_phone` is not started yet is held, and the call returns `true`.
- Re-emits NASH Phone's events under its own name, for example `lb-phone:numberChanged`: on the server `numberChanged`, `phoneNumberGenerated`, `messages:messageSent` and `onAddTransaction`, on the client `phoneToggled`.
- Forwards to the resource named in `CompatConfig.telephone` (`config.lua`), `nash_phone` by default.

{% hint style="warning" %}
Replace an older copy of the compat with the current one. Older copies recorded every app as belonging to the compat itself, so an app's icon stayed on the home screen, with a blank page, after the app's own resource stopped.
{% endhint %}

### What works

| Area | Behaviour |
|---|---|
| Declaration | Every field of an LB declaration is accepted (unused ones are listed under [Declaration fields](#declaration-fields)). `AddCustomApp` returns `true`, or `false` and a reason |
| Live messages | `SendCustomAppMessage` reaches a plain `window` `message` listener, with `event.data` = `{ action, data }` |
| NUI callbacks | `fetch('https://<resource>/<callback>')` from the page reaches the app's own `RegisterNUICallback` |
| Home screen | With `defaultApp = true` the app is installed and its icon placed once per character, on the last page or on a new page when that one is full |
| App Store | The app is listed with its `description`, `developer`, `size` and `images`. With `defaultApp = false` the player gets it there, as on LB Phone |
| Exports | Calls reach NASH Phone's exports of the same name, for example `GetEquippedPhoneNumber` and `SendNotification(source, { app, title, content })` on the server |

### What does not

- **LB's injected globals.** `fetchNui`, `resourceName`, `appName`, `settings`, `getSettings`, `onSettingsChange`, `components` and `createCall` are not put on the page: it is a different origin, and the phone cannot write into it. An app that calls them with no fallback breaks. If you can edit its `index.html`, add the bridge `<script>` tag: it provides all of them except `getSettings`.
- **The resource name, without the bridge.** `resourceName` is never set, so an app that falls back on a hardcoded resource name only works if its folder keeps that name.
- **The top of the screen.** The status bar and the Dynamic Island are painted over the top 62 px of the page. Check that the app's layout leaves room.
- **`GetEquippedPhoneNumber`** returns a number for every player, phone item or not, and creates one if needed. An app that expects `nil` for a player without a phone acts for that player anyway.
- **Server `SendCustomAppMessage`** is forwarded unchanged, so it takes NASH's argument order `(source, identifier, data)`.
- **Messages pushed before the page has loaded** are lost.
- **Exports that are not wired.** Some names get a neutral answer, so the app launches without crashing but without data: `CreateCall`, `IsInCall`, `SetPopUp`, `SetContextMenu`, `ShowComponent`, `SetContactModal`, `ToggleHomeIndicator`. Through the compat, `HasPhone` is answered by `HasPhoneItem`, and the names NASH Phone has no equivalent for on that side (`SendMessage`, `SetSetting` and `AddTransaction` on the client, `SetAppInstalled` on the server) return a neutral value instead of failing. Those actually wired are listed in [Server Exports](server-exports.md) and [Client Exports](client-exports.md).

## Debugging checklist

| Symptom | Check |
|---|---|
| Icon never appears | Check `defaultApp`: with `false` the app waits in the App Store. With `true` it is placed once per character, so a character who removed the icon or deleted the app keeps that choice: look in the App Library (a long press there offers "Add to Home Screen") or the App Store. Then set `Config.CustomAppsSettings.debug = true`: no "ajout" line in the F8 console means the declaration was refused, and the reason is on the console line above |
| App missing from the App Store | Its identifier is already used by a built-in or App Store app. Pick another one |
| Icon appears, page is blank | `ui` does not start with your resource name, or the file is missing from `files{}` in your `fxmanifest.lua` |
| `fetchNui` rejects with "contexte pas encore reçu" | You called it before the phone's context reached the page. Pass your resource name as the third argument of `fetchNui`, or call it later |
| A component resolves instantly with `null` | It is not wired yet. Its name is in the console warning |
| Icon stays after you stop your resource | Should not happen: the phone removes it on `onClientResourceStop`. If it persists, the app was declared by `nash_phone` itself through `Config.CustomApps`, or through an older copy of the LB Phone compat |
| App lost after a `nash_phone` restart | Declared apps are kept in memory only. Declare yours again on `onClientResourceStart` for `nash_phone` |
| LB Phone app: `fetchNui is not defined` or similar in the console | LB's injected globals do not exist here. Add the bridge `<script>` tag to its `index.html` |

## Related

- [Server Events](server-events.md): `nash_phone:customAppUsed`
- [Client Events](client-events.md): `nash-phone:phoneToggled`, state bags
- [Commands](commands.md)
