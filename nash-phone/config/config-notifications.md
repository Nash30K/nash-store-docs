# config/notifications.lua

Clearing **one player's** phone notifications when they join or leave the server.

File: `config/notifications.lua`

This file adds to `Config.Notifications`, so it must stay declared **after** `config/main.lua`
in `fxmanifest.lua`. It completes that table instead of replacing it.

## What is being cleared

The phone's banners: incoming text message, missed call, bank transfer, mail. Each one is stored
in the database under the character who received it. When the player dismisses it from their
lock screen it is **not** deleted: it is marked read and simply leaves the screen.

{% hint style="info" %}
A notification is an **alert**, not the content. The text message stays in Messages, the mail in
Mail, the transfer in Wallet. Purging notifications never deletes anything but the banners
themselves.
{% endhint %}

Notifications are only written for a player who is **online**: nothing piles up in the phone
while they are disconnected. Clearing at session boundaries therefore costs them nothing.

## Settings

```lua
Config.Notifications.purgeOnDrop = true
Config.Notifications.purgeOnJoin = true
Config.Notifications.retentionHours = 48
```

| Key | Type | Default | Description |
|---|---|---|---|
| `purgeOnDrop` | `boolean` | `true` | When the player leaves the server, their notifications are deleted. `false` keeps them waiting for their return |
| `purgeOnJoin` | `boolean` | `true` | The phone starts every session with a clean lock screen. `false` restores whatever was left over |
| `retentionHours` | `number` | `48` | On connect, notifications older than this are deleted. `0` = unlimited |

### Why both purges exist

They are not duplicates. The drop purge does not fire when the **server restarts**, when the
**resource is restarted**, or when a player's session ends abruptly (crash, dropped connection).
The join purge catches all of those cases.

{% hint style="warning" %}
Keep at least one of the two on. With both at `false`, the phone carries its notifications from
session to session and only `retentionHours` still limits what stacks up.
{% endhint %}

### When `retentionHours` matters

Only when `purgeOnJoin` is `false`. With the join purge on, everything has already been deleted
and there is nothing left to date. It becomes your only safety net if you turn both purges off -
for instance on a server where players want yesterday's alerts back. Set the duration you want
then: `24` for a day, `168` for a week.

The cleanup runs **when the player connects**, not in a permanently running task. A server loop
would work around the clock on phones nobody is looking at; the only person this housekeeping
changes anything for is the one who just arrived. It completes before the interface asks for its
notification history, so nothing that was just deleted can still reach the screen.

## Do not confuse with config/main.lua

`Config.Notifications.keepReadDays` and `keepDays`, in
[config/main.lua](config-main.md#notification-retention), sweep the **table**: a periodic pass
across all players, in **days**, whose job is to stop the database growing forever.

This file cleans **one phone**, at the precise moment its owner arrives or leaves. The two
complement each other and can be used together.

| | `config/main.lua` | `config/notifications.lua` |
|---|---|---|
| Scope | The whole table, every player | One character |
| Trigger | One minute after startup, then hourly | Player connect / disconnect |
| Unit | Days | Hours (`retentionHours`) |

## Fallbacks

If this file is deleted or not declared in `fxmanifest.lua`, the server applies
`purgeOnDrop = true`, `purgeOnJoin = true` and `retentionHours = 48`: the same values as
shipped. Without those fallbacks, a missing file would silently disable behaviour the product
announces as active. `nil` and `false` stay distinct: `false` is your explicit choice, `nil`
means the file does not mention the setting.
