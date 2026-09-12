# Client Events

Two families here: **local events** the phone fires on the client so other client scripts can react, and **net events** the server sends to a client. Both are stable integration points; everything else in `client/` is internal plumbing and may change between releases.

{% hint style="warning" %}
Watch the prefix. Local phone events use a **hyphen** (`nash-phone:…`); resource-level net events use an **underscore** (`nash_phone:…`). A handler registered on the wrong spelling never fires and never errors.
{% endhint %}

## Local events (client → client)

Fired with `TriggerEvent` inside the player's own client. Listen with `AddEventHandler`: no `RegisterNetEvent` needed, and no `RegisterNetEvent` wanted (that would let the server spoof them).

<details>

<summary>nash-phone:phoneToggled</summary>

The phone was opened or closed. This is the main hook for anything that must give way while the phone is out.

```lua
AddEventHandler('nash-phone:phoneToggled', function(open)
    if open then
        -- Phone is on screen
    else
        -- Phone is put away
    end
end)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `open` | `boolean` | `true` on open, `false` on close |

**When it fires**

- On every open, including opens triggered by the item, the `ToggleOpen` export or the `nash_phone:open` net event.
- On every close, including the ones the phone forces on itself: the player dies, the phone gets confiscated through `ToggleDisabled`, or the resource stops.

**Ordering:** the event fires **before** the 3D prop is attached. Attaching the prop blocks for up to 3.5 seconds while the model streams in, so the event was deliberately moved ahead of it; do not assume the prop exists in your handler.

The phone itself listens to this event in three places (camera, compass, Snapz map). Yours runs alongside them, in registration order.

</details>

<details>

<summary>nash-phone:setOnScreen</summary>

Fired at the same moment and with the same argument as `nash-phone:phoneToggled`. It exists for scripts written against another phone's API that already listen to this name.

```lua
AddEventHandler('nash-phone:setOnScreen', function(onScreen)
    -- onScreen: boolean
end)
```

Listen to one or the other, not both: you would run your handler twice.

</details>

## Net events (server → client)

<details>

<summary>nash_phone:open</summary>

Opens the phone on that client, unconditionally.

```lua
-- From a server script
TriggerClientEvent('nash_phone:open', src)
```

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `device` | `string?` | Optional. Serial number of the phone item the player opens with. Leave it out from your own scripts |

**Behavior:** calls the same open path as the key binding, so focus, the control lock (the player can still walk), the prop and the state bags all follow. It does **not** check `Config.UseItem`: this is the event the inventory item and the item-gated `nash_phone:requestOpen` both end on, after their own check.

`device` is what relocks a phone that has changed hands (see [Face ID](../apps/README.md#system)): the resource sends it itself from `nash_phone:requestOpen` and the inventory item. Without it, the phone simply opens where it was left.

An open still fails silently if the phone is confiscated (`ToggleDisabled`), the player is dead, or the phone is already open.

</details>

<details>

<summary>nash_phone:toast</summary>

Shows an `ox_lib` notification outside the phone.

```lua
TriggerClientEvent('nash_phone:toast', src, title, text)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `title` | `string` | Notification title |
| `text` | `string` | Notification body |

{% hint style="info" %}
This event is **inert unless `Config.NotifyStyle` is `'oxlib'` or `'both'`**. The default is `'phone'`, where the phone paints its own iOS-style banner instead and this handler returns immediately. If your integration must always be seen, send a real phone notification with the `SendNotification` server export rather than a toast.
{% endhint %}

The phone uses this event itself for one case only: telling a player they do not have the phone item.

</details>

<details>

<summary>nash-phone:hydrate</summary>

Pushes the character's phone identity and settings to the client. Fired on player load and again on every phone bootstrap.

```lua
RegisterNetEvent('nash-phone:hydrate', function(data)
    -- data.number    (string)  the character's phone number
    -- data.settings  (table)   merged settings for that character
end)
```

**Payload**

| Field | Type | Description |
|---|---|---|
| `number` | `string` | Phone number |
| `settings` | `table` | Merged settings (theme, language, profile name, case colour…) |

This is what makes `GetEquippedPhoneNumber` and `GetSettings` answer client-side before the player has ever opened their phone. Reading the state bag `LocalPlayer.state.phoneNumber` is usually simpler than listening here.

</details>

## State bags

Set by the phone and replicated, so they are readable from any resource without an event round-trip. They are populated on player load, **not** on first open: a player who has never taken out their phone still has a readable number.

**On the server**

| Bag | Type | Description |
|---|---|---|
| `Player(src).state.phoneNumber` | `string` | The character's phone number |
| `Player(src).state.phoneName` | `string` | Device name shown in settings. Defaults to `NphoneV1` until the player renames it |

```lua
local number = Player(source).state.phoneNumber
```

**On the client**

| Bag | Type | Description |
|---|---|---|
| `LocalPlayer.state.phoneNumber` | `string` | Own phone number |
| `LocalPlayer.state.phoneOpen` | `boolean` | Phone currently on screen |
| `LocalPlayer.state.flashlight` | `boolean` | Torch currently on |

All three are replicated, so another player's values are readable as `Player(serverId).state.phoneOpen` and so on: useful for scripts that need to know a nearby player has their phone out or their torch lit.

{% hint style="info" %}
`flashlight` is the single source of truth for the torch and it is kept in sync with the drawn light and with the `GetFlashlight` export. Never toggle the light through the camera module directly: use the `ToggleFlashlight` export, or the bag will disagree with what players actually see.
{% endhint %}

## Related

- [Server Events](server-events.md): logging and moderation hooks
- [Commands](commands.md): key binding and the test command set
- [Custom Applications](custom-apps.md): adding your own app to the home screen
