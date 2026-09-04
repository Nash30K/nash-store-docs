# Server Exports

54 exports are declared on the server, in `server/apps/exports.lua`. They are the entry point
for any resource that needs to read a phone number, text a player, drop a notification on
their lock screen, or write a line in their wallet history.

All of them live in the `nash_phone` namespace and are called from a server script:

```lua
local number = exports.nash_phone:GetEquippedPhoneNumber(source)
```

Every value a third-party script passes is bounded before it reaches the database. A title
that is too long for its column is truncated instead of failing the insert, so a call never
throws an SQL error inside your own resource. The exact limits are listed on each export.

{% hint style="warning" %}
19 of these names, plus 10 getters, are **neutral stubs**. They exist so a script written for
another phone does not crash, they return a harmless value, and they change nothing. Each one
is flagged below, and all of them are listed in [Neutral stubs](#neutral-stubs) at the end of
the page.
{% endhint %}

## Identity and ownership

<details>

<summary>GetEquippedPhoneNumber</summary>

Returns the phone number of a player, or of a character that is not connected.

**Signature**

```lua
local number = exports.nash_phone:GetEquippedPhoneNumber(source)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` \| `string` | Player server id, or a framework identifier passed directly as a string (ESX identifier, QBCore citizenid) |

**Returns**

| Name | Type | Description |
|---|---|---|
| `number` | `string?` | Phone number in the `555-0123456` form, or `nil` if the identifier could not be resolved |

**Behavior**

- With a server id, the in-memory directory answers first; if the player is not in it, the
  number is resolved from the framework identifier and read from the database.
- With a string, the value is treated as a framework identifier and the number is **created
  if the character does not have one yet**, which writes a row in `nash_phone_phones`. Use
  this form only for characters you know exist.
- A number is never reassigned, so the result can safely be cached by your resource.

**Example**

```lua
RegisterCommand('mynumber', function(source)
    local number = exports.nash_phone:GetEquippedPhoneNumber(source)
    print(('%s uses %s'):format(GetPlayerName(source), number or 'no phone'))
end, false)
```

</details>

<details>

<summary>GetSourceFromNumber</summary>

Resolves a phone number to the server id of the connected player who owns it.

**Signature**

```lua
local src = exports.nash_phone:GetSourceFromNumber(number)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `number` | `string` | Phone number, exactly as stored (`555-0123456`) |

**Returns**

| Name | Type | Description |
|---|---|---|
| `src` | `number?` | Server id, or `nil` when the number is unknown or its owner is offline |

**Behavior**

- Reads the in-memory directory only, never the database, so it is cheap enough to call in a
  loop.
- The directory is filled when the framework reports the player as loaded, not when the phone
  is first opened: a connected player who has never taken their phone out still resolves.
- A `nil` result means "offline or unknown". It does not distinguish the two.

</details>

<details>

<summary>HasPhoneItem</summary>

Checks whether a player carries the inventory item required to use the phone.

**Signature**

```lua
local hasItem = exports.nash_phone:HasPhoneItem(source)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` | Player server id |

**Returns**

| Name | Type | Description |
|---|---|---|
| `hasItem` | `boolean` | `true` if the player carries the item, or if the item requirement is disabled |

**Behavior**

- Returns `true` immediately when `Config.UseItem = false` in `config/main.lua`.
- Otherwise the inventory bridge is queried for `Config.ItemName` (`'phone'` by default).
- This is the authoritative check. The client export of the same name always returns `true`.

</details>

<details>

<summary>FormatNumber</summary>

Compatibility shim. Returns its argument unchanged.

**Signature**

```lua
local formatted = exports.nash_phone:FormatNumber(number)
```

**Behavior**

Numbers are already stored in their display form (`555-0123456`), so there is nothing to
format. The export exists so a script that pipes every number through `FormatNumber` keeps
working. It is not a validator: whatever you pass comes back, including `nil`.

</details>

<details>

<summary>GetConfig</summary>

Returns the phone configuration, minus the upload credentials.

**Signature**

```lua
local config = exports.nash_phone:GetConfig()
```

**Returns**

| Name | Type | Description |
|---|---|---|
| `config` | `table` | Deep copy of the `Config` table, without the `Upload` key |

**Behavior**

- `Config.Upload` holds the image-hosting API key of the server and is stripped out. Every
  other section is present.
- The copy is deep, so writing into a sub-table does not alter the phone's real
  configuration.
- The copy is built once and the **same instance is handed to every caller**. Treat it as
  read-only: a resource that writes into it changes what the next resource reads.

</details>

<details>

<summary>GetCellTowers</summary>

{% hint style="warning" %}
Stub. Always returns an empty table. nash_phone has no cell tower model, and signal strength
displayed in the status bar is a fixed value.
{% endhint %}

```lua
local towers = exports.nash_phone:GetCellTowers() -- {}
```

</details>

<details>

<summary>ContainsBlacklistedWord</summary>

{% hint style="warning" %}
Stub. Always returns `false`. There is no word filter in the phone; nothing you pass is
inspected. If your server needs message filtering, do it in your own resource before calling
`SendMessage`.
{% endhint %}

```lua
local blocked = exports.nash_phone:ContainsBlacklistedWord(text) -- false
```

</details>

## Settings

<details>

<summary>GetSettings</summary>

Returns the merged settings of the player who owns a phone number.

**Signature**

```lua
local settings = exports.nash_phone:GetSettings(number)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `number` | `string` | Phone **number**, not a server id |

**Returns**

| Name | Type | Description |
|---|---|---|
| `settings` | `table?` | Defaults merged with the player's saved preferences, or `nil` when the number is unknown |

**Notable keys and their defaults**

| Key | Default | Description |
|---|---|---|
| `language` | `Config.Locale` (`'fr'`) | Language of that player's phone |
| `airplane` | `false` | Airplane mode |
| `silent` | `false` | Silent switch |
| `darkMode` | `true` | Dark interface |
| `streamerMode` | `false` | Hides identifying information on screen |
| `hiddenNumber` | `false` | Caller id withheld on outgoing calls |
| `frameColor` | `'cosmic'` | Shell colour, one of `deepblue`, `cosmic`, `silver`, `black` |
| `ringtone` | `'reflection'` | Ringtone id |
| `wallpaper` | `'photo-23'` | Wallpaper id |
| `brightness` | `85` | Screen brightness |
| `phoneScale` | `0.67` | On-screen size of the device |
| `phonePos` | `'br'` | On-screen corner |
| `notifications` | `{ messages = true, phone = true, wallet = true, mail = true }` | Per-application notification switches |
| `passcodeEnabled` | `false` | Lock code enabled |

**Behavior**

- Keys prefixed with an underscore (`_profileName`, `_mutedThreads`, `_earbuds`) are internal
  storage used by other parts of the phone. Do not write logic against them.
- The table also contains `passcode`, the player's lock code in clear. Never forward the
  result of this export to a client.
- A player who has never changed a setting still gets a full table: the defaults are merged
  in before returning.

</details>

<details>

<summary>HasAirplaneMode</summary>

Tells whether the owner of a number has airplane mode on.

**Signature**

```lua
local offline = exports.nash_phone:HasAirplaneMode(number)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `number` | `string` | Phone number |

**Returns**

| Name | Type | Description |
|---|---|---|
| `offline` | `boolean` | `true` when airplane mode is on, `false` when it is off or the number is unknown |

**Behavior**

Worth checking before an interaction that is supposed to behave like a real network. The
phone's own SMS path already honours airplane mode, but the `SendMessage` export does not:
see its block below.

</details>

## Notifications

<details>

<summary>SendNotification</summary>

Pushes a phone notification to one player: stored in the database, shown on the lock screen,
and displayed as a banner.

**Signature**

```lua
-- table form (recommended)
exports.nash_phone:SendNotification(target, { app = ..., title = ..., content = ... })

-- flat form
exports.nash_phone:SendNotification(target, appId, title, text, icon)
```

**Parameters, table form**

| Name | Type | Description |
|---|---|---|
| `target` | `number` \| `string` | Player server id, or a phone number |
| `data.app` | `string?` | Application id the notification belongs to, `'system'` when absent. Max 32 characters |
| `data.title` | `string?` | Bold line. Max 64 characters. Falls back to the localized word for "Phone" in the recipient's own language |
| `data.content` or `data.text` | `string?` | Body. Max 255 characters |
| `data.thumbnail` or `data.avatar` | `string?` | Icon URL. Max 64 characters |

**Parameters, flat form**

| Name | Type | Description |
|---|---|---|
| `target` | `number` \| `string` | Player server id, or a phone number |
| `appId` | `string` | Application id |
| `title` | `string` | Required in this form: a call with no title returns `nil` and sends nothing |
| `text` | `string?` | Body |
| `icon` | `string?` | Icon URL |

**Returns**

| Name | Type | Description |
|---|---|---|
| `ok` | `boolean?` | `true` when the notification was pushed, `nil` when the target could not be resolved or the flat form was missing its title |

**Behavior**

- The target must be **connected**. A phone number whose owner is offline resolves to
  nothing and the call returns `nil`. A raw server id is trusted as given: passing the id of
  a player who has left returns `true` and delivers nothing, so prefer the number form when
  you are not sure the player is still there.
- Each field is truncated on a UTF-8 character boundary to fit its column. The icon is the
  exception: an over-long URL is dropped rather than cut, because a truncated URL points
  nowhere. The interface then falls back to the application icon.
- The notification is written to `nash_phone_notifications` and stays unread until the player
  dismisses it, so it is still on the lock screen when they open the phone later in the
  session. It does **not** cross sessions with the shipped settings: `purgeOnDrop` and
  `purgeOnJoin` are both `true` in `config/notifications.lua`. Send the content to an
  application (an SMS, a mail, a wallet line) when it has to be found again tomorrow, and use
  a notification for the alert itself.
- If the player turned that application's notifications off, the row is still written but no
  banner is shown. Re-enabling the switch brings it back.
- A player whose phone is put away is still told: with `Config.NotifyStyle = 'phone'` (the
  default, in `config/main.lua`) the device rises from the corner where the player parked it
  to show the banner, with `'oxlib'` an `ox_lib` toast is used instead, and `'both'` does the
  two.
- `app` should be an id the phone knows, so the notification inherits the right icon. Common
  ids: `phone`, `messages`, `mail`, `wallet`, `contacts`, `gallery`, `settings`, `services`.
  An unknown id still works, it simply has no icon of its own.

**Example, a shop receipt**

```lua
RegisterNetEvent('my_shop:sold', function(price, label)
    local src = source
    exports.nash_phone:SendNotification(src, {
        app = 'wallet',
        title = 'Purchase',
        content = ('%s, -%d$'):format(label, price),
    })
end)
```

</details>

<details>

<summary>NotifyEveryone</summary>

Sends the same notification to every connected player.

**Signature**

```lua
exports.nash_phone:NotifyEveryone(nil, { app = 'system', title = ..., content = ... })
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| *(first argument)* | `any` | **Ignored.** It only exists so the signature matches the compatibility convention. Pass `nil` |
| `data` | `table` | Same fields and same limits as the table form of `SendNotification` |

**Returns**

Nothing.

**Behavior**

- Only the table form is accepted. A flat call sends nothing.
- Each recipient gets the fallback title in the language of their own phone, not in the
  server language.
- One database row is written per connected player. On a full server that is one insert per
  player, so avoid calling this on a timer.

</details>

<details>

<summary>EmergencyNotification</summary>

Sends a notification flagged as an emergency, and broadcasts a server event so job scripts
can react.

**Signature**

```lua
exports.nash_phone:EmergencyNotification(target, { title = ..., content = ... })
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `target` | `number` \| `string` | Player server id, or a phone number |
| `data` | `table` | Same fields and limits as `SendNotification` |

**Returns**

| Name | Type | Description |
|---|---|---|
| `ok` | `boolean` | Always `true` |

**Behavior**

- Defaults differ from `SendNotification`: the application is `phone` and the fallback title
  is the localized word for "Emergency", again in the recipient's language.
- The server event `nash-phone:emergency` is fired with the resolved server id (or the raw
  target when it could not be resolved) and the data table. **The event fires even when the
  notification could not be delivered**, which is what makes this export usable as a
  dispatch trigger.

**Example, mirroring emergencies into a dispatch resource**

```lua
-- `target` is who the notification was addressed to: a server id when the player is
-- connected, the raw value you passed otherwise.
AddEventHandler('nash-phone:emergency', function(target, data)
    TriggerEvent('my_dispatch:alert', target, data.title, data.content or data.text)
end)
```

</details>

## Contacts

<details>

<summary>AddContact</summary>

Writes a contact into a player's address book.

**Signature**

```lua
exports.nash_phone:AddContact(ownerNumber, { number = ..., name = ..., avatar = ... })
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `ownerNumber` | `string` | Phone number of the player **whose** address book is written |
| `data.number` | `string` | Number of the contact being added. Required, max 16 characters |
| `data.name` | `string?` | Display name, max 64 characters. Falls back to the number |
| `data.avatar` | `string?` | Avatar URL, max 255 characters |

**Returns**

Nothing. The write is synchronous but silent.

**Behavior**

- Nothing happens if the owner number is unknown or `data.number` is missing.
- The pair (owner, number) is unique in `nash_phone_contacts`: adding a contact that already
  exists **updates** the name and the avatar instead of failing.
- The Contacts application loads its list when it opens, so a contact added while the app is
  already on screen appears the next time the player opens it.

**Example, a company handing out its number**

```lua
local shopNumber = '555-0000001'
RegisterNetEvent('my_job:hire', function()
    local src = source
    local number = exports.nash_phone:GetEquippedPhoneNumber(src)
    if not number then return end
    exports.nash_phone:AddContact(number, { number = shopNumber, name = 'Dispatch' })
end)
```

</details>

## Messages

<details>

<summary>SendMessage</summary>

Sends an SMS from any number to any number, and delivers it live when the recipient is
online.

**Signature**

```lua
local payload = exports.nash_phone:SendMessage(from, to, message, attachments, cb)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `from` | `string` | Sender number, max 16 characters. Does not have to belong to a player |
| `to` | `string` | Recipient number, max 16 characters |
| `message` | `string` | Body, truncated at 16000 characters |
| `attachments` | `any` | **Ignored.** Present for signature compatibility |
| `cb` | `function?` | Called with the payload once the message is written, or with `nil` on refusal |

**Returns**

| Name | Type | Description |
|---|---|---|
| `payload` | `table?` | `{ messageId, sender, recipient, message, createdAt }`, or `nil` when a required field was empty |

**Behavior**

- The row is written to `nash_phone_messages` with type `text` regardless of whether the
  recipient is connected, so an offline player finds the message when they log back in.
- The server event `nash-phone:messages:messageSent` is fired with the payload. The same
  event is fired by the phone itself when a player sends an SMS, which is how you catch
  replies.
- Live delivery only happens for a connected recipient.

{% hint style="warning" %}
This export bypasses two things the in-app SMS path honours: the recipient's **airplane
mode**, and their **muted conversations**. A message sent through this export is delivered
and notified in both cases. Check `HasAirplaneMode` first if you want a realistic behaviour.
Note as well that the banner shows the raw sender number, not the contact name, because the
sender is not necessarily a person.
{% endhint %}

**Example, a shop line that answers**

```lua
local SHOP = '555-0000001'

-- Confirmation sent to the customer
exports.nash_phone:SendMessage(SHOP, customerNumber, 'Your order is ready.')

-- Replies sent back to that number
AddEventHandler('nash-phone:messages:messageSent', function(data)
    if data.recipient ~= SHOP then return end
    print(('%s answered: %s'):format(data.sender, data.message))
end)
```

Without that listener nothing reads messages sent to a number that belongs to no player:
the row is written and no one is notified.

</details>

<details>

<summary>SendCoords</summary>

Sends a position as a text message.

**Signature**

```lua
local payload = exports.nash_phone:SendCoords(from, to, coords)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `from` | `string` | Sender number |
| `to` | `string` | Recipient number |
| `coords` | `table?` | Table with `x` and `y`. Missing values are sent as `0` |

**Returns**

| Name | Type | Description |
|---|---|---|
| `payload` | `table?` | The payload returned by `SendMessage` |

**Behavior**

- The body is built from the `location_shared` locale string, in the **server** language
  (`Config.Locale`), not in the recipient's. It reads `Location shared (x, y)` in English.
- The message is a plain text message. The phone does not turn it into a map pin.
- Nothing is sent when `from` or `to` is missing.

</details>

## Wallet

{% hint style="danger" %}
`AddTransaction` and `SentMoney` write **history rows only**. They move no money. The
balances the wallet displays are read live from the framework (bank and cash); the
`nash_phone_bank` table is a log. Call your framework or your banking resource to actually
move the funds, then call these exports if you want the line to appear in the phone.
{% endhint %}

<details>

<summary>AddTransaction</summary>

Appends one line to a player's wallet history.

**Signature**

```lua
exports.nash_phone:AddTransaction(number, amount, title, image)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `number` | `string` | Phone number of the account holder |
| `amount` | `number` | Signed amount. Positive writes an incoming line, negative an outgoing one |
| `title` | `string?` | Label of the operation |
| `image` | `string?` | Passed to the event only. Never stored |

**Returns**

Nothing.

**Behavior**

- Nothing happens if the number is unknown.
- The amount is rounded towards zero and clamped to a signed 32-bit integer. A value that is
  not a number becomes `0`; the call never raises an arithmetic error inside your script.
- The title is written into two columns of different sizes: truncated to **16 characters**
  for the counterpart shown in the list, and to **128 characters** for the memo. A long
  label is therefore visible in full only in the detail view.
- The server event `nash-phone:onAddTransaction` is fired with `(dir, number, amount, title,
  image)`, where `dir` is `'in'` or `'out'` and the title is the **untruncated** one.
- If the player is connected, the line is pushed to the open phone immediately.

**Example, logging a paycheck**

```lua
RegisterNetEvent('my_job:paid', function(amount)
    local src = source
    local number = exports.nash_phone:GetEquippedPhoneNumber(src)
    if not number then return end
    -- The money itself was already moved by your framework call.
    exports.nash_phone:AddTransaction(number, amount, 'Salary')
end)
```

</details>

<details>

<summary>SentMoney</summary>

Writes the two sides of a transfer in one call.

**Signature**

```lua
exports.nash_phone:SentMoney(from, to, amount)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `from` | `string` | Number debited in the history |
| `to` | `string` | Number credited in the history |
| `amount` | `number` | Amount, same clamping as `AddTransaction` |

**Returns**

Nothing.

**Behavior**

- Two rows are written: an incoming line on `to` and an outgoing line on `from`.
- Both are labelled with the `tx_transfer` locale string in the **server** language
  (`Transfer` in English).
- A side whose number is unknown is skipped silently, so a transfer to a non-player number
  writes one row instead of two.
- No money is moved. See the warning above.

</details>

## Calls

<details>

<summary>GetCall</summary>

Returns the live state of a call by its id.

**Signature**

```lua
local call = exports.nash_phone:GetCall(callId)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `callId` | `number` | Call id |

**Returns**

| Name | Type | Description |
|---|---|---|
| `call` | `table?` | The call record, or `nil` if that id is unknown |

**Call fields**

| Field | Type | Description |
|---|---|---|
| `id` | `number` | Call id |
| `channel` | `number` | Voice channel used by the call |
| `callerSrc` / `calleeSrc` | `number` | Server ids of both sides |
| `callerNumber` / `calleeNumber` | `string` | Numbers of both sides |
| `callerName` / `calleeName` | `string?` | How each side is displayed to the other |
| `answeredAt` | `number?` | Timestamp of the answer, `nil` while ringing |
| `video` | `boolean` | `true` when the caller started a video call |
| `appId` | `string` | Application the call was placed from, `'phone'` by default |
| `guests` | `table` | Conference participants |
| `speakers` / `listeners` | `table` | Speakerphone bookkeeping |
| `_done` | `boolean` | `true` once the call is over |

{% hint style="warning" %}
This returns the **live internal table**, not a copy. Writing into it corrupts a call in
progress. Read it, do not modify it, and do not persist its shape: it is internal and can
change between releases.
{% endhint %}

</details>

<details>

<summary>IsInCall</summary>

Tells whether a player is currently in a call.

**Signature**

```lua
local busy = exports.nash_phone:IsInCall(source)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` | Player server id |

**Returns**

| Name | Type | Description |
|---|---|---|
| `busy` | `boolean` | `true` if the player takes part in a call that has not ended |

**Behavior**

Covers ringing calls, answered calls and conference guests. The client export of the same
name is a stub and always returns `false`: ask the server.

</details>

<details>

<summary>EndCall</summary>

Hangs up whatever call a player is in.

**Signature**

```lua
local ended = exports.nash_phone:EndCall(source)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` | Player server id |

**Returns**

| Name | Type | Description |
|---|---|---|
| `ended` | `boolean` | `true` if a call was found and terminated, `false` otherwise |

**Behavior**

- The other side is notified and the voice channel is released for everyone involved.
- This is the export to pair with the client `ToggleDisabled`: confiscating a phone kicks the
  player out of the voice channel, but the call itself stays open server-side until it is
  ended here.

**Example, confiscating a phone properly**

```lua
RegisterNetEvent('my_prison:jail', function()
    local src = source
    exports.nash_phone:EndCall(src)
    TriggerClientEvent('my_prison:disablePhone', src, true) -- calls ToggleDisabled(true)
end)
```

</details>

<details>

<summary>CreateCall</summary>

{% hint style="warning" %}
Stub. Always returns `0` and starts nothing. Calls are placed from the phone interface; there
is no server-side way to force one. `GetCall`, `IsInCall` and `EndCall` above are real, so
you can observe and end calls, just not start them.
{% endhint %}

```lua
local id = exports.nash_phone:CreateCall(...) -- 0
```

</details>

## Mail

<details>

<summary>GetEmailAddress</summary>

Returns the mail address of a character.

**Signature**

```lua
local address = exports.nash_phone:GetEmailAddress(source)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` | Player server id |

**Returns**

| Name | Type | Description |
|---|---|---|
| `address` | `string?` | The address stored in `nash_phone_mail_accounts`, or `nil` |

**Behavior**

- The address belongs to the character and is created the first time that character opens the
  Mail application.
- `nil` therefore means "this player has never opened Mail", which is a real answer and not a
  placeholder. If your script needs an address to exist, ask the player to open the app once.

</details>

## Callback relays

These four exports are thin wrappers around `ox_lib` callbacks. They exist so a script
written against another phone's callback API keeps working. Calling `lib.callback` directly
is strictly equivalent, and is what the phone does internally.

<details>

<summary>RegisterCallback / BaseCallback</summary>

Registers a server callback. The two names are aliases of the same function.

**Signature**

```lua
exports.nash_phone:RegisterCallback(name, handler)
exports.nash_phone:BaseCallback(name, handler)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `name` | `string` | Callback name |
| `handler` | `function` | Handler, receives `source` then the client arguments |

**Behavior**

Equivalent to `lib.callback.register(name, handler)`. The callback is registered under the
name you give, in the global `ox_lib` namespace: pick a name prefixed with your own resource
to avoid a collision.

</details>

<details>

<summary>TriggerClientCallback</summary>

Calls a client callback and receives the answer in a function.

**Signature**

```lua
exports.nash_phone:TriggerClientCallback(name, source, cb, ...)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `name` | `string` | Callback name registered on the client |
| `source` | `number` | Player server id |
| `cb` | `function` | Receives the client's return values |
| `...` | `any` | Arguments forwarded to the client |

**Behavior**

Equivalent to `lib.callback(name, source, cb, ...)`.

</details>

<details>

<summary>AwaitClientCallback</summary>

Calls a client callback and waits for the answer.

**Signature**

```lua
local result = exports.nash_phone:AwaitClientCallback(name, source, ...)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `name` | `string` | Callback name registered on the client |
| `source` | `number` | Player server id |
| `...` | `any` | Arguments forwarded to the client |

**Returns**

Whatever the client handler returns.

**Behavior**

Equivalent to `lib.callback.await(name, source, ...)`. It yields, so it must be called from
inside a thread or an event handler, never at file load.

</details>

## Neutral stubs

Everything listed here is declared, callable, and does nothing. None of them writes to the
database, none of them raises an error, and none of them logs a warning. They exist only so
that a script written for another phone runs unmodified.

<details>

<summary>No-op stubs (return nil)</summary>

| Export | What a script written for another phone expects | What happens here |
|---|---|---|
| `AirShare` | Share a file between phones | Nothing |
| `AddCheck` | Give a bank cheque | Nothing |
| `RemoveCheck` | Remove a cheque | Nothing |
| `FactoryReset` | Wipe a phone | Nothing. Player data stays in place |
| `ResetSecurity` | Clear the lock code | Nothing. The code lives in the `passcode` setting |
| `SaveBattery` | Persist a battery level | Nothing. Battery is a client-side display value |
| `SaveAllBatteries` | Persist every battery level | Nothing |
| `ToggleVerified` | Verify a social account | Nothing |
| `ChangePassword` | Change a social account password | Nothing |
| `DeleteBirdyAccount` | Delete a Twitter-like account | Nothing |
| `DeleteInstaPicAccount` | Delete a photo-network account | Nothing |
| `DeleteTrendyAccount` | Delete a short-video account | Nothing |
| `SendDarkChatMessage` | Post in an anonymous chat | Nothing |
| `SendDarkChatLocation` | Post a position in an anonymous chat | Nothing |
| `CreateDarkChatChannel` | Create a channel | Nothing |
| `DeleteDarkChatChannel` | Delete a channel | Nothing |
| `AddUserToDarkChatChannel` | Add a member | Nothing |
| `RemoveUserFromDarkChatChannel` | Remove a member | Nothing |
| `AddCustomCoin` | Add a crypto currency | Nothing |

</details>

<details>

<summary>Getter stubs (fixed return value)</summary>

| Export | Returns | Note |
|---|---|---|
| `GetCellTowers` | `{}` | No cell tower model |
| `ContainsBlacklistedWord` | `false` | No word filter |
| `GetSocialMediaUsername` | `nil` | Social accounts exist, with their own tables, but are not exposed here |
| `IsVerified` | `false` | No verification flag reachable from this API |
| `PostBirdy` | `false` | The application is real; posting from another resource is not supported |
| `GetBirdyPost` | `nil` | Same |
| `GetPin` | `nil` | The lock code is the `passcode` key of the settings, and is not exposed as a getter |
| `IsPhoneDead` | `false` | Battery is a client-side display value. The client export of the same name is real |
| `CreateCall` | `0` | Calls start from the interface |
| `AddCrypto` | `false` | No crypto wallet backend |
| `RemoveCrypto` | `false` | Same |
| `GetCoin` | `nil` | Same |

</details>

{% hint style="info" %}
The stub list is what is true today. If a feature you need is stubbed, the phone's own
backend usually covers it internally; what is missing is the public entry point. Ask on the
Discord before building a workaround on top of a stub, since a stub that becomes real keeps
its name and signature.
{% endhint %}
