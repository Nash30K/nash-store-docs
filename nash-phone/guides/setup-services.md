# Wire your jobs into the Services app

Let citizens call or message the police, EMS, a mechanic or a taxi from the phone, and let the
players holding those jobs answer from their own Services screen.

## Prerequisites

- A framework whose job names you know exactly (`police`, `ambulance`, `mechanic`…)
- Everything happens in `config/services.lua`
- Optionally, a square PNG per service to drop into the `service_icons/` folder

## How it works

A citizen opens the Services app, picks a business, and either calls it or sends a written
request. Every employee of that job who is **on duty and accepting requests** gets a phone
notification and sees the request in their queue. One of them takes it, the citizen is
notified, and the employee closes it when the job is done.

The job a player holds is read from the framework and **never** from the interface: that is
the single line that stops a modified client from declaring itself police.

## 1. Declare your businesses

The **order of `list` is the display order**. To move Taxi above Mechanic, move its block -
there is no `order` field to keep in sync.

```lua
-- config/services.lua
Config.Services = {
    enabled = true,
    requestTimeout = 15,
    historyKeep = 25,
    cooldown = 20,
    ownDuty = true,

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
        {
            job = 'taxi',
            label = 'Taxi',
            place = 'Taxi HQ',
            icon = 'taxi.png',
            accent = '#ffd60a',
            canCall = true,
            canMessage = true,
            multiResponders = false,
            shareLocation = true,
        },
    },
}
```

| Field | Type | Description |
| ----- | ---- | ----------- |
| `job` | `string` | **Exact** job name on the framework side. It is the only thing the server looks at to know who works here, and it must match character for character |
| `label` | `string` | Displayed name. It is **not** translated: "Police" stays "Police" in both languages, like a proper noun. Leave it empty to use the phone's own translation. Truncated to 48 characters |
| `place` | `string` | Location shown under the name. Decorative |
| `icon` | `string` | Image filename in the resource's `service_icons/` folder |
| `accent` | `string` | Colour of the badge, in hexadecimal |
| `canCall` | `boolean` | The citizen can call this service |
| `canMessage` | `boolean` | The citizen can send it a written request |
| `multiResponders` | `boolean` | `true`: several employees can answer the same request. The first one takes it, the others see "Taken by X" and can join: what you want for police and EMS. `false`: first come, first served, the request disappears for the others: what you want for a taxi or a mechanic: one job, one customer |
| `shareLocation` | `boolean` | Offers the citizen the option to attach their position. They remain free to decline: this setting only opens the possibility |

{% hint style="warning" %}
A business missing from `list` does not exist in the app. A player holding the matching job
will therefore not see their "My service" page either.

A duplicate `job` is ignored: the first block declaring a given job wins.
{% endhint %}

## 2. Add the icons

Drop a PNG into the `service_icons/` folder at the root of the resource, then write its
filename in the `icon` field. Nothing to rebuild.

- Square PNG, transparent background, 128 × 128 or 256 × 256 pixels.
- The phone rounds the corners itself: do not round them in the image.
- **A URL does not work.** The phone runs in an embedded browser with no network access at
  all: an `https://` icon will never load, even though it displays fine in your own browser.
- **Do not put the file in `web/build/`.** That folder is rewritten on every phone update and
  your image would disappear.

A missing file breaks nothing: the service falls back to a badge in its `accent` colour with
its initial. You can add a service first and its icon later.

Restart the resource after adding a file: the images are served by the game, not by the
phone, so it has to read them again.

## 3. Choose who owns the duty state

```lua
ownDuty = true,
```

| Value | Behaviour |
| ----- | --------- |
| `true` (shipped) | The app's own Duty switch is the truth. The phone keeps the state in memory and gives it back to the player when they reconnect. Works on any server |
| `false` | The phone hides the switch and waits for **another** script to tell it who is on duty, through `exports.nash_phone:SetServiceDuty(source, true)`. Use this if you already have a locker room or a clock-in system and want a single source of truth |

{% hint style="danger" %}
With `ownDuty = false` and nothing calling the export, **every service stays "Unavailable"**.
That is not a bug: no employee was ever declared on duty.
{% endhint %}

### Driving duty from your own script

```lua
-- server side, from your locker room / clock-in resource
exports.nash_phone:SetServiceDuty(source, true)         -- on duty, accepting requests
exports.nash_phone:SetServiceDuty(source, true, false)  -- on duty, NOT accepting requests
exports.nash_phone:SetServiceDuty(source, false)        -- off duty
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `src` | `number` | Player server id |
| `on` | `boolean` | On duty or not |
| `receive` | `boolean?` | Accepting requests. Left out, the previous value is kept (`true` on the first call) |

It returns `false` when the player has no job, or a job that is not in `list`.

To read the headcount from another script:

```lua
local onDuty = exports.nash_phone:GetServiceOnDuty('police')  -- number
```

"Reachable" means **on duty AND accepting requests:** both, not either.

## 4. Tune the general settings

| Key | Default | Effect |
| --- | ------- | ------ |
| `cooldown` | `20` | Minimum seconds between two requests from the same player. Without it, a single citizen can flood a whole precinct. `0` disables the limit |
| `historyKeep` | `25` | How many of the most recent requests the employee's "Recent requests" list shows. It is a display limit: older rows stay in `nash_phone_service_requests`, they are simply no longer listed |
| `requestTimeout` | `15` | Minutes before an unanswered request closes itself. **Read from the config but not applied in the current release**: a request stays `pending` until an on-duty employee takes it or closes it |
| `enabled` | `true` | `false` removes every service from the app |

## 5. Restart

```
restart nash_phone
```

The list is read once at resource start and cleaned up: a field that is absent or of the wrong
type is dropped rather than allowed to break a callback three hours later.

## How to know it works

1. Open Services as a citizen. Your businesses are listed, in the order of `list`, each with
   its icon and its live headcount.
2. Connect a second player, give them the matching job, and flip the Duty switch on their
   "My service" page. The citizen's headcount goes from 0 to 1 without reopening the app.
3. Send a request from the citizen. The employee gets a phone notification -
   *"&lt;name&gt; needs your help."*: and the request appears in their queue.
4. Take the request on the employee side. The citizen gets *"&lt;name&gt; is responding to your
   request."* and sees the state change in their own request list, which shows their five most
   recent requests.
5. Check the two `multiResponders` behaviours with two employees on duty: on a `true` service
   both can join; on a `false` service the second one is told the request is already taken.
6. In the database:

    ```sql
    SELECT id, job, name, status, taker_name, created_at
    FROM nash_phone_service_requests
    ORDER BY id DESC LIMIT 10;
    ```

    `status` goes `pending` → `accepted` → `closed`.

### If a service is always "Unavailable"

- The `job` string does not match the framework exactly. This is by far the most common cause.
- Nobody is on duty, or the on-duty employees turned off "accept requests".
- `ownDuty = false` and no script ever calls `SetServiceDuty`.
