# Developer API

nash_phone exposes its API through resource exports, server events broadcast to other
resources, and player state bags. This section documents what other scripts are allowed to
call, what each call actually does, and what it returns.

## Sections

| Page | Contents |
| ---- | -------- |
| [Server Exports](server-exports.md) | 60 exports declared on the server: numbers, notifications, contacts, SMS, wallet history, calls, mail, Services duty, first-run setup |
| [Client Exports](client-exports.md) | 73 exports declared on the client: opening the phone, applications, notifications, flashlight, battery, orientation, shell colour |
| [Client Events](client-events.md) | Events fired on the player's client, net events, state bags |
| [Server Events](server-events.md) | Hooks fired on the server when something happens inside the phone |
| [Commands](commands.md) | Player commands, key bindings, the server console command |
| [Custom Applications](custom-apps.md) | Adding your own app: `AddCustomApp`, the iframe bridge, components |

## Calling convention

An export is only visible on the side that declares it. Server exports are called from a
server script, client exports from a client script.

```lua
-- server script
local number = exports.nash_phone:GetEquippedPhoneNumber(source)

-- client script
exports.nash_phone:OpenApp('messages')
```

A few names exist on both sides with different signatures (`GetEquippedPhoneNumber`,
`GetPhoneNumber`, `HasPhoneItem`, `GetConfig`, `GetSettings`, `SendNotification`, `AddContact`,
`SendCustomAppMessage`, `IsInCall`, `IsPhoneDead`). Read the page for the side you are calling from before assuming the
behaviour, because two of those pairs disagree on purpose:

| Export | Server | Client |
| ------ | ------ | ------ |
| `HasPhoneItem` | Real inventory check | Always `true`, ownership is validated server-side |
| `IsPhoneDead` | Stub, always `false` | Real, `true` when the displayed battery is 0 |

## Phone numbers

Numbers are strings built from `Config.Phone` in `config/main.lua`: the prefix (`'555'` by
default), a hyphen, then `digits` digits (`7` by default), which gives `555-0123456`.

One number is assigned per framework identifier and is never reassigned. The resource never
changes or deletes a number once its row exists in `nash_phone_phones` (the only later write
to that table is an internal bookkeeping flag), so a number you cache in your own script stays
valid for the life of the character.

The hyphen is load-bearing: several server exports accept either a server id or a phone
number in the same parameter, and they tell them apart by looking for that hyphen. The
generator always inserts it, even with `prefix = ''` (which gives `-0123456`), so never strip
it from a number before passing it back to the phone.

## State bags

The resource publishes a few values as state bags, which is often cheaper than an export
call, especially in a loop.

| Bag | Side | Value |
| --- | ---- | ----- |
| `Player(src).state.phoneNumber` | Server | The player's phone number, or `nil` |
| `Player(src).state.phoneName` | Server | Device name from the player's profile, `'NphoneV1'` when unset |
| `LocalPlayer.state.phoneNumber` | Client | The local player's phone number |
| `LocalPlayer.state.phoneOpen` | Client | `true` while the phone is on screen, replicated to other clients |
| `LocalPlayer.state.flashlight` | Client | `true` while the torch is on, replicated to other clients |

`phoneNumber` and `phoneName` are set when the framework reports the player as loaded, not
when the phone is first opened, so they are already filled for a player who has never taken
their phone out.

{% hint style="info" %}
Only the server writes `phoneNumber` and `phoneName`: if a client changes one of them on its own
state bag, the server puts the real value back right away.
{% endhint %}

## Compatibility exports

A large part of this API reproduces the export names and signatures of the most widespread
FiveM phone. The intent is that a third-party script written for that phone runs against
nash_phone after changing the resource name and nothing else.

Those compatibility exports fall into two groups:

- **Implemented.** The call is wired to the nash_phone backend and does what its name says.
- **Neutral stub.** The export exists so the call does not error, and returns a harmless
  value (`nil`, `false`, `0`, `{}`). It changes nothing in the phone.

{% hint style="warning" %}
Stubs are the single most expensive thing to misread in this section. A stub never fails,
never logs, and never tells you it did nothing, so a script built on one appears to work
until someone checks the phone. Every stub is flagged with a warning box on its page, and
both pages end with a full list of them.
{% endhint %}

Feature by feature, here is what is actually behind the compatibility names:

| Area | Server | Client |
| ---- | ------ | ------ |
| Numbers, identity, settings | Implemented | Implemented |
| Notifications | Implemented | Implemented |
| Contacts | Implemented | Implemented |
| SMS | Implemented | Not exposed |
| Wallet history | Implemented, history rows only, no money moved | Not exposed |
| Calls | `GetCall`, `IsInCall` and `EndCall` implemented, `CreateCall` is a stub | Stubs |
| Phone shell, torch, battery, orientation | Not exposed | Implemented |
| Social networks, dark chat, crypto, cell towers, security PIN | Stubs | Stubs |

The social applications and the lock code are not missing from the phone. They exist with
their own backend and their own database tables; they simply have no public export in this
API, and the compatibility names that would reach them on another phone are inert here.
There is no crypto wallet in NASH Phone at all: every crypto name is a stub on both sides.

## Reading the API pages

Each export gets its own collapsible block with its signature, its parameters, its return
values and its behaviour. Blocks are grouped by area, and the neutral stubs are gathered at
the end of each page in a single table so you can check a name in one look.
