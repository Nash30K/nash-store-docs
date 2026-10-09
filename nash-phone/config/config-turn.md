# config/turn.lua

The TURN relay for video: FaceTime, video calls from Phone and Contacts, and InstaPic lives.
**Strongly recommended on a public server:** without a relay, some players get the sound of a
call but no picture (4G box, CGNAT, strict NAT, VPN, or two players behind the same box). In an
InstaPic live they get neither, since the live's sound travels with the picture.

File: `config/turn.lua`

```lua
Config.VideoTurn = {
    provider = '',
    keyId = '',
    apiToken = '',
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `provider` | `string` | `''` | `'cloudflare'` turns the relay on. Any other value, empty included, leaves it off |
| `keyId` | `string` | `''` | The Key ID of your Cloudflare TURN key |
| `apiToken` | `string` | `''` | The API token of that same key |

The relay stays off until all three are filled in. Spaces around the values are ignored.

{% hint style="danger" %}
**This file is server-side only.** It is declared in `server_scripts` in `fxmanifest.lua`, right
after `config/upload.lua`, and it must stay there. Never move it into `shared_scripts` or
`client_scripts`: a shared script is downloaded into the cache of every player who connects, and
your API token would be public. The `GetConfig` server export leaves this block out too.
{% endhint %}

## Getting a key

1. Create a free account on [https://dash.cloudflare.com](https://dash.cloudflare.com)
2. In the dashboard, open **Realtime**, then **TURN**, and create a key
3. Paste its Key ID into `keyId` and its API token into `apiToken`, and set
   `provider = 'cloudflare'`
4. Restart the resource

Cloudflare's free tier covers the first 1,000 GB each month, then $0.05 per GB (Cloudflare
pricing). Only calls that cannot connect directly go through the relay.

## What the server console says

| Line | Meaning |
| ---- | ------- |
| `[nash_phone] video: Cloudflare TURN relay ready.` | The relay works |
| `[nash_phone] video: Cloudflare TURN request failed (HTTP <code>), check keyId and apiToken in config/turn.lua. Video calls fall back to STUN only.` | Cloudflare refused the request or could not be reached. Printed once per series of failures; the server tries again on its own, from one minute up to one hour later |
| `[nash_phone] video: Cloudflare TURN relay is available again.` | The relay works again after a failure |
| `[nash_phone] video: no TURN relay configured, ... See config/turn.lua.` | Printed at startup when video calls are on and no relay is set, neither here nor in `Config.VideoCall.iceServers` |

## How it works

The API token never leaves your server. The server asks Cloudflare for short-lived relay
credentials, valid 4 hours and renewed when less than 2 hours remain, and only those are sent
to players, with each video call and each live. The Cloudflare servers are added after the
entries of `Config.VideoCall.iceServers`.

## Related settings

The video settings themselves (resolution, frame rate, framing, STUN servers, your own TURN
server) are in [config/main.lua](config-main.md#video-calls-facetime). The full walkthrough,
with how to read the F8 traces, is in
[Video calls and live streams](../guides/setup-video-calls.md).
