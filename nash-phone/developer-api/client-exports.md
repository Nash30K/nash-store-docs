# Client Exports

67 exports are declared on the client, in `client/modules/exports.lua`. They cover everything
that happens on the player's own screen: opening the phone, taking it away, opening an
application, the torch, the battery indicator, the orientation and the colour of the device
held in hand.

All of them live in the `nash_phone` namespace and are called from a client script:

```lua
exports.nash_phone:OpenApp('messages')
```

Client exports act on the local player only. Anything that concerns another player, or that
must survive a reconnection, belongs to the [server exports](server-exports.md).

{% hint style="warning" %}
24 of these names, plus 8 getters, are **neutral stubs**, and one implemented export
(`SetAppHidden`) currently has no effect. Each case is flagged below, and all of them are
listed in [Neutral stubs](#neutral-stubs) at the end of the page.
{% endhint %}

## Opening and state

<details>

<summary>ToggleOpen</summary>

Opens, closes or toggles the phone.

**Signature**

```lua
local isOpen = exports.nash_phone:ToggleOpen(open)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `open` | `boolean?` | `true` opens, `false` closes, `nil` toggles |

**Returns**

| Name | Type | Description |
|---|---|---|
| `isOpen` | `boolean` | State of the phone **after** the call |

**Behavior**

- Opening is refused, and `false` is returned, when the phone was taken away with
  `ToggleDisabled`, or when the player is dead or cuffed.
- Closing always succeeds and returns `false`.
- Opening gives the interface keyboard and mouse focus while keeping game input alive, so the
  player can still walk with the phone in hand.

</details>

<details>

<summary>IsOpen / IsPhoneOnScreen</summary>

Two names for the same value: is the phone currently on screen.

**Signature**

```lua
local open = exports.nash_phone:IsOpen()
local open = exports.nash_phone:IsPhoneOnScreen()
```

**Returns**

| Name | Type | Description |
|---|---|---|
| `open` | `boolean` | `true` while the phone is displayed |

**Behavior**

The same information is replicated as a state bag, `LocalPlayer.state.phoneOpen`, which is
also readable from other clients. Prefer the state bag inside a per-frame loop.

</details>

<details>

<summary>IsDisabled</summary>

Tells whether the phone was taken away.

**Signature**

```lua
local disabled = exports.nash_phone:IsDisabled()
```

**Returns**

| Name | Type | Description |
|---|---|---|
| `disabled` | `boolean` | `true` when `ToggleDisabled(true)` is in effect |

**Behavior**

The flag lives in the client session only. It is reset when the resource restarts or the
player reconnects: if your script needs a confiscation to survive a reconnect, re-apply it
when the player loads.

</details>

<details>

<summary>ToggleDisabled</summary>

Takes the phone away, or gives it back. This is the export for prison, cuffs, death, or a
no-signal zone.

**Signature**

```lua
exports.nash_phone:ToggleDisabled(disabled)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `disabled` | `boolean` | `true` takes the phone away, `false` gives it back |

**Returns**

Nothing.

**Behavior**

A phone that has been taken away:

- no longer opens by **any** path: the command, the key bind, the inventory item, the
  `nash_phone:open` event, and the `ToggleOpen` / `OpenApp` exports all go through the same
  guard;
- no longer rings on an incoming call, and displays nothing;
- no longer shows notifications: the client `SendNotification` returns `false` immediately;
- no longer lights anything: the torch is switched off and cannot be switched back on.

If the phone is open when the call is made, it closes immediately.

{% hint style="warning" %}
A call in progress is only half stopped here. The player leaves the voice channel, but the
call itself stays open on the server and the other party keeps waiting. To hang up as well,
call the server export `EndCall(source)` from your server script.
{% endhint %}

**Example, prison**

```lua
-- client
RegisterNetEvent('my_prison:disablePhone', function(state)
    exports.nash_phone:ToggleDisabled(state)
end)
```

```lua
-- server
RegisterNetEvent('my_prison:jail', function()
    local src = source
    exports.nash_phone:EndCall(src)
    TriggerClientEvent('my_prison:disablePhone', src, true)
end)
```

</details>

<details>

<summary>ReloadPhone</summary>

Asks the interface to fetch its data again.

**Signature**

```lua
exports.nash_phone:ReloadPhone()
```

**Returns**

Nothing.

**Behavior**

- The page re-runs its bootstrap request, which re-reads the number, the settings, the home
  screen layout and the rest of the player's data from the server.
- The web page itself is **not** reloaded and the phone is not opened or closed. This is a
  data refresh, not a restart.
- Useful after your resource has changed something in the database directly and wants the
  open phone to reflect it.

</details>

## Identity

<details>

<summary>GetEquippedPhoneNumber / GetPhoneNumber</summary>

Two names for the local player's phone number.

**Signature**

```lua
local number = exports.nash_phone:GetEquippedPhoneNumber()
local number = exports.nash_phone:GetPhoneNumber()
```

**Returns**

| Name | Type | Description |
|---|---|---|
| `number` | `string?` | `555-0123456`, or `nil` before the server has hydrated the phone |

**Behavior**

The value is pushed by the server when the player loads, and mirrored into the state bag
`LocalPlayer.state.phoneNumber`, which other clients can read as well.

</details>

<details>

<summary>HasPhoneItem</summary>

{% hint style="warning" %}
Always returns `true` on the client. Item ownership is checked server-side, where a client
cannot lie about it. Use the server export of the same name for a real answer.
{% endhint %}

```lua
local hasItem = exports.nash_phone:HasPhoneItem() -- true
```

</details>

<details>

<summary>FormatNumber</summary>

Compatibility shim. Returns its argument unchanged, exactly like the server export of the
same name.

```lua
local formatted = exports.nash_phone:FormatNumber(number)
```

</details>

<details>

<summary>GetConfig</summary>

Returns the client configuration.

**Signature**

```lua
local config = exports.nash_phone:GetConfig()
```

**Returns**

| Name | Type | Description |
|---|---|---|
| `config` | `table` | The `Config` table as loaded on the client |

**Behavior**

- This is the **live table**, not a copy. Unlike the server export, nothing is stripped and
  nothing is duplicated: writing into it changes the behaviour of the phone for that player
  until the resource restarts. Read it, do not modify it.
- `Config.Upload` does not exist on the client. The upload credentials are loaded from a
  server-only file and are never sent to a player.

</details>

## Notifications

<details>

<summary>SendNotification</summary>

Shows a notification on the local phone, without going through the server.

**Signature**

```lua
exports.nash_phone:SendNotification({ app = ..., title = ..., content = ... })
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `data.app` | `string?` | Application id, `'system'` when absent |
| `data.title` | `string?` | Bold line |
| `data.content` or `data.text` | `string?` | Body |
| `data.thumbnail` or `data.avatar` | `string?` | Icon URL |

**Returns**

| Name | Type | Description |
|---|---|---|
| `ok` | `boolean` | `false` when the phone was taken away, `true` otherwise |

**Behavior**

- Purely local: nothing is written to the database, so the notification disappears on
  reconnect and no other player is aware of it.
- The interface still applies the player's own filters: an application whose notification
  switch is off produces nothing at all, and Do Not Disturb keeps the entry in the list
  without showing it on screen.

{% hint style="warning" %}
With the default `Config.NotifyStyle = 'phone'`, this export raises **two** notifications for
one call: the one you asked for, plus a second one filed under the `phone` application. That
is a known duplication of this compatibility export. Two alternatives declared in
`client/modules/notifications.lua` push exactly one: `PushNotification(appId, title, text,
icon)` for an entry inside the phone, and `Notify(title, text, kind)` for a toast outside it.
For anything that should survive a reconnection, use the server export instead.
{% endhint %}

</details>

## Applications

<details>

<summary>OpenApp</summary>

Opens the phone and jumps straight into an application.

**Signature**

```lua
local ok = exports.nash_phone:OpenApp(app, data)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `app` | `string` | Application id |
| `data` | `any?` | **Transmitted but ignored.** The interface does not read a payload today |

**Returns**

| Name | Type | Description |
|---|---|---|
| `ok` | `boolean` | `false` when the phone refused to open (taken away, dead, cuffed), `true` otherwise |

**Behavior**

- The phone is opened first. If it refuses, the application order is not sent at all,
  otherwise the app would open behind an invisible screen and appear at the next legitimate
  unlock.
- The lock screen is dismissed: opening an application this way does not ask for the lock
  code.
- The id is not checked against the installed list. A store application opens even if the
  player never installed it, and an unknown id opens a placeholder screen.

**Preinstalled ids**

`phone`, `messages`, `camera`, `wallet`, `contacts`, `gallery`, `mail`, `notes`, `maps`,
`weather`, `clock`, `calendar`, `facetime`, `music`, `safari`, `services`, `health`,
`reminders`, `appstore`, `settings`, `passwords`

**App Store ids**

`snake`, `tictactoe`, `calculator`, `stocks`, `compass`, `ticktok`, `instapic`, `playtube`,
`snapz`, `chatapp`, `birdby`

**Example, a job dispatch opening the Services app**

```lua
RegisterNetEvent('my_job:dispatch', function()
    exports.nash_phone:OpenApp('services')
end)
```

</details>

<details>

<summary>CloseApp</summary>

Returns to the home screen.

**Signature**

```lua
exports.nash_phone:CloseApp()
```

**Returns**

Nothing.

**Behavior**

Closes the application that is open and clears its navigation stack. The phone itself stays
on screen; use `ToggleOpen(false)` to put it away.

</details>

<details>

<summary>SetAppInstalled</summary>

Installs or uninstalls an application for the local player.

**Signature**

```lua
exports.nash_phone:SetAppInstalled(app, installed)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `app` | `string` | Application id |
| `installed` | `boolean?` | Anything other than `false` installs |

**Returns**

Nothing.

**Behavior**

- Installing adds the id to the player's installed list and drops the icon on the last home
  page, unless it is already somewhere on the home screen or in the dock.
- Uninstalling removes the icon from the home screen, from the dock and from any folder, and
  closes the application if it was open.
- The installed list is saved server-side, so the change survives a reconnection.
- This is the export for an application your server hands out as a reward or restricts to a
  job: use it with an App Store id, since preinstalled applications are not part of the
  installed list.

</details>

<details>

<summary>SetAppHidden</summary>

{% hint style="warning" %}
Declared, callable, and currently without effect. The export sends its message to the
interface, but no handler reads it, so nothing is hidden and nothing is reported. To take an
icon off the home screen today, use `SetAppInstalled(app, false)`.
{% endhint %}

```lua
exports.nash_phone:SetAppHidden(app, hidden) -- no effect
```

</details>

## Contacts

<details>

<summary>AddContact</summary>

Adds a contact to the local player's address book.

**Signature**

```lua
local sent = exports.nash_phone:AddContact({ number = ..., firstname = ..., lastname = ..., avatar = ... })
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `data.number` | `string` | Contact number. Required |
| `data.firstname` | `string?` | First name |
| `data.lastname` | `string?` | Last name |
| `data.avatar` | `string?` | Avatar URL |

**Returns**

| Name | Type | Description |
|---|---|---|
| `sent` | `boolean` | `false` when `number` is missing, `true` when the request was sent |

**Behavior**

- The display name is the first and last name joined by a space, trimmed. When both are
  empty, the number is used as the name.
- The request goes to the server, which upserts the row: adding a number that is already in
  the book updates the existing entry instead of creating a duplicate.
- `true` means "the request left", not "the contact was written". The server's answer is
  discarded.
- The Contacts application reads its list when it opens, so a contact added while the app is
  already on screen shows up the next time it is opened.

</details>

## Gallery

<details>

<summary>SaveToGallery</summary>

Saves an image URL into the player's camera roll.

**Signature**

```lua
exports.nash_phone:SaveToGallery(link)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `link` | `string` | Image URL |

**Returns**

Nothing. The server's answer is discarded.

**Behavior**

- The URL is validated server-side: it must start with `http://` or `https://`, or be a
  relative path with no scheme, no `..` and no leading `//`, and it must stay under 1024
  characters. Anything else is rejected silently.
- The photo lands in the `Camera` album, like a picture taken with the phone.
- A player is limited to 20 saved captures per 60 seconds, this export included.

</details>

## Battery

The battery is a display value. Nothing in the phone drains it, and a phone at 0 % still
opens and works normally. These exports exist so a server that wants a battery mechanic can
drive the indicator from its own resource.

<details>

<summary>GetBattery</summary>

```lua
local level = exports.nash_phone:GetBattery() -- 0 to 100, 100 by default
```

</details>

<details>

<summary>SetBattery</summary>

Sets the battery level shown in the status bar.

**Signature**

```lua
exports.nash_phone:SetBattery(level)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `level` | `number` | Clamped between 0 and 100. A value that is not a number becomes 100 |

**Behavior**

- The status bar is refreshed immediately, and the value is kept: the phone's own periodic
  refresh, every 15 seconds while it is open, reads the same variable instead of overwriting
  it with 100.
- The level is not stored anywhere. It is back to 100 after a resource restart or a
  reconnection, so a battery mechanic has to persist it in its own resource.

</details>

<details>

<summary>IsCharging</summary>

```lua
local charging = exports.nash_phone:IsCharging() -- false by default
```

</details>

<details>

<summary>ToggleCharging</summary>

Shows or hides the charging indicator.

**Signature**

```lua
exports.nash_phone:ToggleCharging(charging)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `charging` | `boolean` | Anything other than `true` turns the indicator off |

**Behavior**

Cosmetic only: the level does not go up on its own while charging is displayed.

</details>

<details>

<summary>IsPhoneDead</summary>

```lua
local dead = exports.nash_phone:IsPhoneDead() -- battery <= 0
```

**Behavior**

Reads the displayed battery level. A `true` here does **not** prevent the phone from opening:
if you want a dead battery to lock the device, pair it with `ToggleDisabled(true)`. The
server export of the same name is a stub and always returns `false`.

</details>

## Flashlight

<details>

<summary>ToggleFlashlight</summary>

Switches the torch on or off.

**Signature**

```lua
exports.nash_phone:ToggleFlashlight(on)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `on` | `boolean` | `true` switches on, anything else switches off |

**Returns**

Nothing.

**Behavior**

- This is a real light in front of the player, the same beam as the camera flash, not just an
  icon.
- The state is replicated as `LocalPlayer.state.flashlight`, so other clients can read it.
- The Control Center icon is updated too, which keeps the button in sync with the beam.
- A phone that has been taken away can only be switched **off**: a call with `true` is
  ignored.

{% hint style="info" %}
Calling this with no argument switches the torch off. It does not toggle. Read `GetFlashlight`
and pass the opposite if you want a toggle.
{% endhint %}

</details>

<details>

<summary>GetFlashlight</summary>

```lua
local on = exports.nash_phone:GetFlashlight() -- boolean
```

**Behavior**

Always agrees with the beam, the state bag and the Control Center button: every path that
changes the torch goes through the same entry point.

</details>

## Screen orientation

<details>

<summary>ToggleLandscape</summary>

Lays the phone on its side.

**Signature**

```lua
local landscape = exports.nash_phone:ToggleLandscape(on)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `on` | `boolean?` | `true` lays it down, `false` stands it up, `nil` toggles |

**Returns**

| Name | Type | Description |
|---|---|---|
| `landscape` | `boolean` | State after the call |

**Behavior**

The interface tracks one token per requester. This export owns its own token, the Camera
application owns another, so a script that stands the phone back up does not interrupt a
player who is filming, and vice versa.

</details>

<details>

<summary>IsLandscape</summary>

```lua
local landscape = exports.nash_phone:IsLandscape()
```

**Behavior**

Reports the token owned by this export only. It reads `false` while the Camera application
holds the phone sideways with its own token.

</details>

## Device colour

<details>

<summary>SetPhoneVariation</summary>

Changes the colour of the device held in hand.

**Signature**

```lua
local ok = exports.nash_phone:SetPhoneVariation(variation)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `variation` | `string?` | `deepblue`, `cosmic` (default), `silver` or `black`. `nil` restores the player's own colour |

**Returns**

| Name | Type | Description |
|---|---|---|
| `ok` | `boolean` | `false` when the colour is unknown, or when `Config.Prop.skin.enabled` is `false` |

**Behavior**

- Purely visual and local. The player's saved setting is **not** overwritten, so a script that
  imposes a colour for a scene does not steal it permanently.
- The colours are defined in `Config.Prop.skin.colors` in `config/main.lua`. Adding one there
  requires a matching back-panel image, and the same id has to exist in the settings screen
  and in the server whitelist.
- The texture swap is local to the player: on their screen every phone takes that colour,
  including other players'. This is a known limit of the method.

</details>

<details>

<summary>GetPhoneVariation</summary>

```lua
local variation = exports.nash_phone:GetPhoneVariation() -- 'cosmic' by default
```

**Behavior**

Returns the colour the player **asked for**: their saved setting when it is a valid id, the
configured default otherwise. It is not a report of what is currently painted on the prop, so
it does not reflect a temporary colour set by `SetPhoneVariation`.

</details>

## Settings

<details>

<summary>GetSettings</summary>

Returns the local player's settings, as cached on the client.

**Signature**

```lua
local settings = exports.nash_phone:GetSettings()
```

**Returns**

| Name | Type | Description |
|---|---|---|
| `settings` | `table` | The settings pushed by the server, or an empty table before hydration |

**Behavior**

- The table is filled when the server hydrates the phone at player load. Calling this at
  resource start, before the framework reports the player, gives an empty table.
- The keys are the ones listed in the server `GetSettings` block.
- This is the live table, not a copy. Writing into it desynchronises the phone from what the
  server has stored.

</details>

<details>

<summary>GetAirplaneMode</summary>

```lua
local airplane = exports.nash_phone:GetAirplaneMode() -- false by default
```

</details>

<details>

<summary>GetStreamerMode</summary>

```lua
local streamer = exports.nash_phone:GetStreamerMode() -- false by default
```

**Behavior**

Streamer mode masks the player's own number where the phone displays it. Read it if your own
resource shows that number on screen and should hide it too.

</details>

## Callback relays

Three wrappers around `ox_lib` callbacks, kept for compatibility. Calling `lib.callback`
directly is equivalent.

<details>

<summary>RegisterClientCallback</summary>

Registers a client callback.

```lua
exports.nash_phone:RegisterClientCallback(name, handler)
```

Equivalent to `lib.callback.register(name, handler)`. The name lands in the global `ox_lib`
namespace, so prefix it with your own resource name.

</details>

<details>

<summary>TriggerCallback</summary>

Calls a server callback and receives the answer in a function.

```lua
exports.nash_phone:TriggerCallback(name, cb, ...)
```

Equivalent to `lib.callback(name, false, cb, ...)`.

</details>

<details>

<summary>AwaitCallback</summary>

Calls a server callback and waits for the answer.

```lua
local result = exports.nash_phone:AwaitCallback(name, ...)
```

Equivalent to `lib.callback.await(name, false, ...)`. It yields, so call it from inside a
thread or an event handler.

</details>

## Neutral stubs

Declared, callable, and without effect. They exist so a script written for another phone runs
unmodified. None of them errors, logs, or changes anything on screen.

<details>

<summary>No-op stubs (return nil)</summary>

| Export | What a script written for another phone expects | What happens here |
|---|---|---|
| `ToggleHomeIndicator` | Show or hide the home bar | Nothing |
| `SetServiceBars` | Force the signal bars | Nothing. Signal is a fixed value |
| `EnableWalkableCam` | Camera mode that lets the player walk | Nothing |
| `DisableWalkableCam` | Leave that mode | Nothing |
| `ToggleSelfieCam` | Switch to the front camera | Nothing. The Camera application has its own selfie mode |
| `ToggleCameraFrozen` | Freeze the camera | Nothing |
| `SetPopUp` | Display a modal inside the phone | Nothing |
| `SetContextMenu` | Display a context menu | Nothing |
| `ShowComponent` | Display an interface component | Nothing |
| `SetCameraComponent` | Overlay something on the camera | Nothing |
| `SetContactModal` | Open the add-contact sheet | Nothing. Use `AddContact` |
| `CreateCall` | Place a call | Nothing. Calls start from the interface |
| `CreateCustomNumber` | Register a virtual number | Nothing |
| `RemoveCustomNumber` | Remove one | Nothing |
| `CreateDynamicCustomNumber` | Register a virtual number with a handler | Nothing |
| `RemoveDynamicCustomNumber` | Remove one | Nothing |
| `EndCustomCall` | End a call with a virtual number | Nothing |
| `PostBirdy` | Post on the Twitter-like network | Nothing. The application is real, posting from outside is not exposed |
| `SendCompanyMessage` | Message from a company account | Nothing |
| `SendCompanyCoords` | Position from a company account | Nothing |
| `ToggleCompanyCalls` | Take or drop company calls | Nothing |
| `AddCheck` | Give a bank cheque | Nothing |
| `RemoveCheck` | Remove a cheque | Nothing |
| `GetCellTowers` | List cell towers | Nothing. Stubbed on both sides |

</details>

<details>

<summary>Getter stubs (fixed return value)</summary>

| Export | Returns | Note |
|---|---|---|
| `IsWalkingCamEnabled` | `false` | No walkable camera mode |
| `IsSelfieCam` | `false` | The Camera application tracks its own mode internally |
| `IsInCall` | `false` | **Always**, even during a real call. Ask the server export `IsInCall(source)` |
| `IsLive` | `false` | Live streams exist in the photo network but are not exposed here |
| `GetCompanyCallsStatus` | `false` | No company call system |
| `GetCoinValue` | `0` | No crypto market |
| `GetCryptoWallet` | `{}` | Same |
| `GetOwnedCoin` | `false` | Same |

</details>

{% hint style="info" %}
Custom applications are not part of this list. `AddCustomApp`, `RemoveCustomApp`,
`SendCustomAppMessage` and `GetCustomApps` are implemented in
`client/modules/custom_apps.lua` and let another resource add its own application to the
phone. They are documented separately in this section.
{% endhint %}
