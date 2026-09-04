# config/services.lua

The businesses a citizen can reach from the Services app, and how their employees answer.

File: `config/services.lua`, loaded as a shared script.

**The order of `list` is the order displayed.** To move Taxi above Mechanic, move its block:
there is no `order` field to keep in sync.

A business absent from this list does not exist in the app. A player holding the matching job
will not see their "My service" page either.

{% hint style="info" %}
This file is read **once** when the resource starts. Run `restart nash_phone` after editing it.
{% endhint %}

## General settings

```lua
Config.Services = {
    enabled = true,
    requestTimeout = 15,
    historyKeep = 25,
    cooldown = 20,
    ownDuty = true,
    list = { },
}
```

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `enabled` | `true` | `false` loads no service at all, so the app has nothing to show. To remove the icon entirely, disable `services` in [config/apps.lua](config-apps.md) |
| `requestTimeout` | `15` | Minutes before an unanswered request closes itself and turns "expired", so a citizen is not left waiting on a service that emptied out |
| `historyKeep` | `25` | Requests kept in a business's "recent requests" history. Older ones are deleted. This is a log book, not an archive |
| `cooldown` | `20` | Minimum seconds between two requests from the same player. `0` disables the limit. Without it a single citizen can drown a whole squad |
| `ownDuty` | `true` | Whether the phone manages the on-duty state itself. See below |

### ownDuty

| Value | Behaviour |
| ----- | --------- |
| `true` (shipped) | The app's Duty switch is the source of truth. The phone keeps the state in memory and hands it back to the player when they reconnect. Works on any server |
| `false` | The switch disappears from the app and the phone waits for another script to tell it who is on duty, through `exports.nash_phone:SetServiceDuty(source, on, receive)` |

Use `false` if you already have a duty system (a locker room, a time clock) and want a single
source of truth.

{% hint style="warning" %}
With `ownDuty = false` and nobody calling the export, **every service stays "unavailable"**.
That is not a fault: no employee was ever declared on duty.
{% endhint %}

Two read/write exports exist for that case:

| Export | Side | Description |
| ------ | ---- | ----------- |
| `exports.nash_phone:SetServiceDuty(src, on, receive)` | Server | Puts a player on or off duty. `receive` is optional and controls whether they get new requests. Returns `false` if the player has no job matching an entry of `list` |
| `exports.nash_phone:GetServiceOnDuty(job)` | Server | Read-only count of how many people are reachable for that job right now |

"Reachable" means **on duty and accepting requests**, both at once. That is the whole
difference between the two switches in the app.

## Business fields

```lua
list = {
    {
        job = 'police',
        label = 'Police',
        place = 'Mission Row',
        icon = 'police.png',
        accent = '#0a84ff',
        canCall = true,
        canMessage = true,
        multiResponders = true,
        shareLocation = true,
    },
}
```

| Field | Default if absent | Description |
| ----- | ----------------- | ----------- |
| `job` | required | The **exact** job name on the framework side. This is the only thing the server looks at to know who works here, so it must match character for character. Comparison is strict: no case folding, no trimming |
| `label` | the `job` value | Displayed name. It is **not** translated: "Police" stays "Police" in both languages, like a proper noun. Leave it empty to use the phone's own translation. Max 48 characters |
| `place` | none | Location shown under the name. Decorative. Max 48 characters |
| `icon` | none | Image file in the resource's `service_icons/` folder. Max 64 characters |
| `accent` | none | Colour of the chip, in hexadecimal. Max 16 characters |
| `canCall` | `true` | The citizen can call this service |
| `canMessage` | `true` | The citizen can send it a written request |
| `multiResponders` | **`false`** | See below |
| `shareLocation` | `true` | Offers the citizen the option of attaching their position. They stay free to refuse: this only opens the possibility |

{% hint style="warning" %}
`multiResponders` is the one field that defaults to `false` when you leave it out, while
`canCall`, `canMessage` and `shareLocation` default to `true`. Write it explicitly on police
and EMS entries.
{% endhint %}

| `multiResponders` | Behaviour |
| ----------------- | --------- |
| `true` | Several employees can answer the same request. The first takes it, the others see "Taken by X" and can join. This is what you want for police and EMS |
| `false` | First come, first served. The request disappears for everyone else. This is what you want for a taxi or a mechanic: one job, one customer |

A duplicate `job` is ignored: the first entry wins.

## Icons

Drop your PNG into the resource's `service_icons/` folder and write its filename in `icon`.
Nothing has to be rebuilt: the folder is declared in `fxmanifest.lua` as `service_icons/*.png`
and the interface reads the file directly.

A missing file does not show a broken image: the app falls back to a coloured chip using
`accent`.

## What lives where

- **In memory:** who is on duty. That is a session state. A server that restarts has nobody
  connected, therefore nobody on duty, so persisting it would only return a false truth to the
  first player who comes back.
- **In the database:** the requests, in `nash_phone_service_requests`. An employee has to find
  their log book again after a restart, and a citizen has to be able to see their request was
  taken.

Nothing is taken on trust from the interface. The NUI never states which job a player holds or
whether they are on duty: it asks. The job comes from the framework bridge and from nothing
else, which is what stops a modified client from declaring itself a police officer.

A call placed from the Services app shows the service `label` on the other side, never the
personal number of whoever picks up.

Attaching a position sends a real map pin into the Messages app, clickable and settable as a
GPS waypoint, not a sentence of text.

## Requirements

The job must be readable through the framework bridge. On a custom framework, `job(src)` in
`server/bridge/custom.lua` must return the documented shape, otherwise the Services app
recognises no employee. See [config/bridge.lua](config-bridge.md).
