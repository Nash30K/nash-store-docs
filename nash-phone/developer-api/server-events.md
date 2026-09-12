# Server Events

Server-side hooks fired by NASH Phone with `TriggerEvent`. Listen to them from your own server script with `AddEventHandler` to log, moderate, or react to what happens inside the phone.

They are fire-and-forget. Nothing in the phone waits for a handler, no handler can cancel the action that fired it, and several of them are wrapped in `pcall` on the phone side (a transfer is already booked when `nash-phone:onAddTransaction` fires: an error in your handler will not roll it back).

{% hint style="warning" %}
Two prefixes are in use and they are not interchangeable. Phone domain events use a **hyphen** (`nash-phone:…`); resource-level events use an **underscore** (`nash_phone:…`). An `AddEventHandler` on the wrong spelling is silent: no error, no call.
{% endhint %}

All timestamps (`createdAt`) are Unix epoch **milliseconds** with one-second precision (`os.time() * 1000`).

## Social networks

The three events below exist for third-party resources: logging, moderation, Discord relays. Nothing inside the phone consumes them.

<details>

<summary>nash-phone:social:posted</summary>

A player published a post in one of the social apps.

```lua
AddEventHandler('nash-phone:social:posted', function(data)
    print(('[%s] %s posted #%d'):format(data.appId, data.author, data.postId))
end)
```

**Payload:** one table

| Field | Type | Description |
|---|---|---|
| `appId` | `string` | `snapz`, `instapic`, `birdby` or `ticktok` |
| `postId` | `number` | Row id in `nash_phone_social_posts` |
| `author` | `string` | In-app username of the poster (not the phone number) |
| `createdAt` | `number` | Epoch ms |

**Limits:** the payload does not carry the body or the media URL. Read them back from `nash_phone_social_posts` if your handler needs the content:

```lua
local row = MySQL.single.await(
    'SELECT `body`, `media_url`, `place` FROM `nash_phone_social_posts` WHERE `id` = ?',
    { data.postId })
```

The test command `/phonefakepost` fires the same event with a seeded author, so a moderation log will show entries that no real player wrote. See [Commands](commands.md).

</details>

<details>

<summary>nash-phone:social:storyPosted</summary>

A player published a story.

```lua
AddEventHandler('nash-phone:social:storyPosted', function(data)
    -- data.appId, data.storyId, data.author, data.createdAt
end)
```

**Payload:** one table

| Field | Type | Description |
|---|---|---|
| `appId` | `string` | `snapz`, `instapic` or `chatapp` |
| `storyId` | `number` | Row id in `nash_phone_social_stories` |
| `author` | `string` | In-app username |
| `createdAt` | `number` | Epoch ms |

**Limits:** the story kind (`image`, `video`, `text`), its media URL and its text are not in the payload; read them from `nash_phone_social_stories`. Stories expire from the feed after 24 hours but the rows stay, so a handler that queries later still finds them.

</details>

<details>

<summary>nash-phone:social:messageSent</summary>

A private message was sent inside a social app (the DM, not the SMS app).

```lua
AddEventHandler('nash-phone:social:messageSent', function(data)
    -- data.appId, data.messageId, data.sender, data.recipient
    -- data.type, data.share, data.text, data.imageUrl, data.createdAt
end)
```

**Payload:** one table

| Field | Type | Description |
|---|---|---|
| `appId` | `string` | `chatapp`, `snapz`, `instapic`, `birdby` or `ticktok` |
| `messageId` | `number` | Row id in `nash_phone_social_messages` |
| `sender` | `string` | In-app reference of the sender |
| `recipient` | `string` | In-app reference of the recipient |
| `type` | `string?` | `post` for a shared publication, `nil` for plain text |
| `share` | `table?` | Snapshot of the shared publication when `type == 'post'`, otherwise `nil` |
| `text` | `string?` | Message body: `nil` when the message is a shared publication |
| `imageUrl` | `string?` | Attached image, `nil` if none |
| `createdAt` | `number` | Epoch ms |

**Note:** `text` keeps its original meaning (plain text only); `type` and `share` were added on top and are always absent for a text message. On `chatapp`, the in-app reference is the phone number; on the other four it is the account username.

The event fires whether or not the recipient is online. An offline recipient reads the message on their next connection: the row is written either way.

</details>

## Messages and mail

<details>

<summary>nash-phone:messages:messageSent</summary>

An SMS was written to `nash_phone_messages`. Fired both by the Messages app and by the `SendMessage` server export.

```lua
AddEventHandler('nash-phone:messages:messageSent', function(data)
    -- data.messageId, data.sender, data.recipient, data.message
    -- data.mtype, data.imageUrl, data.createdAt
end)
```

**Payload:** one table

| Field | Type | Description |
|---|---|---|
| `messageId` | `number` | Row id in `nash_phone_messages` |
| `sender` | `string` | Sender phone number |
| `recipient` | `string` | Recipient phone number |
| `message` | `string` | Body: the text, or the serialized coordinates for a location |
| `mtype` | `string?` | `text`, `image` or `location` |
| `imageUrl` | `string?` | Image URL for `mtype == 'image'` |
| `createdAt` | `number` | Epoch ms |

{% hint style="info" %}
`mtype` and `imageUrl` are only present when the message comes from the Messages app. Messages injected through `exports.nash_phone:SendMessage(...)` always land as plain text and omit both fields. Test with `data.mtype or 'text'`.
{% endhint %}

**Airplane mode:** the event fires even when the recipient has airplane mode on. The row is written and only the live delivery is skipped, exactly like a real phone.

</details>

<details>

<summary>nash-phone:mail:sent</summary>

A mail was sent from the Mail app to another character's address.

```lua
AddEventHandler('nash-phone:mail:sent', function(data)
    -- data.mailId, data.sender, data.senderName
    -- data.recipient, data.recipientName, data.subject, data.body, data.createdAt
end)
```

**Payload:** one table

| Field | Type | Description |
|---|---|---|
| `mailId` | `number` | Row id of the copy delivered to the recipient's inbox |
| `sender` | `string` | Sender mail address |
| `senderName` | `string` | Sender display name |
| `recipient` | `string` | Recipient mail address |
| `recipientName` | `string` | Recipient display name |
| `subject` | `string` | Subject, possibly empty |
| `body` | `string` | Full body |
| `createdAt` | `number` | Epoch ms |

**Note:** a mail is stored twice (one row in the sender's `sent` folder, one in the recipient's `inbox`). `mailId` is the **inbox** row; the `sent` copy has a different id.

</details>

## Wallet

<details>

<summary>nash-phone:onAddTransaction</summary>

A line was written to the phone's wallet history. Fired by an in-app transfer between two players and by the `AddTransaction` / `SentMoney` server exports.

```lua
AddEventHandler('nash-phone:onAddTransaction', function(direction, number, amount, label, image)
    -- direction: 'in' | 'out'
end)
```

**Parameters:** positional, not a table

| Name | Type | Description |
|---|---|---|
| `direction` | `string` | `in` (credit) or `out` (debit) |
| `number` | `string` | Phone number of the account the line belongs to |
| `amount` | `number` | Absolute amount, always positive |
| `label` | `string` | Label supplied by the caller, **untruncated** |
| `image` | `string?` | Optional thumbnail, `nil` for an in-app transfer |

**Note:** a player-to-player transfer fires the event **twice:** once with `out` for the sender, once with `in` for the recipient. Deduplicate on `(number, direction)` if you keep an external ledger.

`label` is the value the caller passed, in full. The columns it is stored in are shorter (`counterpart` is `VARCHAR(16)`, `memo` is `VARCHAR(128)`), so what you receive here can be longer than what a later `SELECT` returns.

</details>

## Identity and lifecycle

<details>

<summary>nash-phone:numberChanged</summary>

Fired on every phone bootstrap, once the player's number is known and their state bags are set.

```lua
AddEventHandler('nash-phone:numberChanged', function(source, number)
    -- source: number, number: string
end)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` | Player server ID |
| `number` | `string` | The character's phone number |

**Note:** despite the name this fires on every bootstrap, not only when the number actually changes. It is the reliable signal for "this player's phone number is now readable". The same value is on the state bag `Player(source).state.phoneNumber`.

A bootstrap runs once when the character is loaded, before the phone is ever opened, then again at each opening. Expect this event several times per session, and keep your handler safe to run more than once.

</details>

<details>

<summary>nash-phone:phoneNumberGenerated</summary>

Fired **only** the first time a number is created for a character: the true "new phone" signal.

```lua
AddEventHandler('nash-phone:phoneNumberGenerated', function(source, number)
    -- Give a starter item, write a welcome SMS, log the new line…
end)
```

**Parameters:** same as `nash-phone:numberChanged`.

**Order:** `numberChanged` fires first, then `phoneNumberGenerated`. A handler on both will see both for a brand new character.

**When:** as soon as the character is loaded by your framework, not when the player first opens the phone. It fires exactly once per character, even across server restarts.

**Created while offline:** a number can be created for a character who is not connected, for example when another resource calls an export with that character's identifier. The event then fires at that character's next connection, with a valid `source`, so a starter item or a welcome message still reaches them.

</details>

<details>

<summary>nash-phone:setupCompleted</summary>

The player finished the first-run setup (language, country, appearance, cloud address, Face ID, passcode).

```lua
AddEventHandler('nash-phone:setupCompleted', function(source, data)
    -- data.owner, data.language, data.country
    -- data.appearance, data.cloudEmail, data.faceId
end)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` | Player server ID |
| `data` | `table` | See below |

**`data` fields**

| Field | Type | Description |
|---|---|---|
| `owner` | `string` | Framework identifier of the character |
| `language` | `string?` | Chosen locale code |
| `country` | `string?` | Chosen country |
| `appearance` | `string?` | `light` or `dark` |
| `cloudEmail` | `string?` | Cloud account address entered during setup |
| `faceId` | `boolean` | Face ID enabled |

**Note:** the passcode is deliberately **not** in the payload. Setup writes everything at once at the end, so this event fires exactly once per character; a player who abandons midway leaves no half-configured phone and fires nothing.

Setup can be replayed for a character with `Setup.reinitialiser(owner)` server-side, or with the `/phonesetupreset` test command.

</details>

<details>

<summary>nash-phone:deviceChanged</summary>

The player opened a **different handset** from the one their setup was done on. The setup row has just been reset, so `nash-phone:setupCompleted` will fire again once they finish.

```lua
AddEventHandler('nash-phone:deviceChanged', function(source, data)
    -- data.owner, data.device
end)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `source` | `number` | Player server ID |
| `data` | `table` | See below |

**`data` fields**

| Field | Type | Description |
|---|---|---|
| `owner` | `string` | Framework identifier of the character |
| `device` | `string` | Serial of the new handset, now recorded for this character |

**Note:** this does **not** fire the first time a serial is read on a phone that already existed. That one is adopted silently, so an update does not wake the setup up for every player on the server. Only a genuinely different device fires it.

It also never fires with `Config.UseItem = false`, or on an inventory that cannot keep data per item instance: there is nothing to compare. See [Custom Inventory](../compatibility/custom-inventory.md).

</details>

## Emergency

<details>

<summary>nash-phone:emergency</summary>

Fired by the `EmergencyNotification` server export, so police / EMS resources can pick up a call for help.

```lua
AddEventHandler('nash-phone:emergency', function(target, data)
    -- target: player server ID, or the raw value passed if it could not be resolved
    -- data:   the table given to EmergencyNotification
end)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `target` | `number` \| `string` | Server ID of the notified player. Falls back to the value originally passed to the export when it could not be resolved (offline player, unknown number) |
| `data` | `table` | The notification table: typically `app`, `title`, `content`, `thumbnail` |

{% hint style="warning" %}
Always type-check `target`. It is a `number` in the normal case, but stays a `string` phone number when the target is offline: and the notification is then not delivered to anyone, only the event fires.
{% endhint %}

</details>

## Custom applications

<details>

<summary>nash_phone:customAppUsed</summary>

A player opened a custom app. Note the **underscore** prefix on this one.

```lua
AddEventHandler('nash_phone:customAppUsed', function(src, identifier)
    if identifier ~= 'my_app' then return end
    -- Server-side reaction to the app being opened
end)
```

**Parameters**

| Name | Type | Description |
|---|---|---|
| `src` | `number` | Player server ID |
| `identifier` | `string` | App identifier as declared in `AddCustomApp` / `Config.CustomApps` |

**Why it matters:** an app declared at runtime from a client script cannot ship an `onServerUse` callback (a Lua function does not cross the network). This event is the supported server-side entry point for those apps. Only an app declared in `config/custom_apps.lua` gets its `onServerUse` called directly.

The event fires for **every** custom app, including ones from other resources, so always filter on `identifier`. See [Custom Applications](custom-apps.md).

</details>

## Net events registered on the server

<details>

<summary>nash_phone:requestOpen</summary>

Client → server request to open the phone, gated by the inventory item.

```lua
TriggerServerEvent('nash_phone:requestOpen')
```

**Parameters:** none.

**Behavior**

- When `Config.UseItem = true`, the server checks that the player owns `Config.ItemName` through the inventory bridge. Without it, the player gets a `nash_phone:toast` and the phone stays closed.
- On success the server fires `nash_phone:open` back at that client.

This is the path used by both the `/phone` command and the key binding, which is why the item check cannot be bypassed client-side. From a **client** script, prefer the `ToggleOpen` export: it also honours the "phone confiscated" state. From a **server** script, use `TriggerClientEvent('nash_phone:open', src)` to open a phone unconditionally.

</details>
