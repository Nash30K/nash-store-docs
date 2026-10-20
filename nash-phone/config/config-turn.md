# config/turn.lua

The TURN relay for video: FaceTime, video calls from Phone and Contacts, and InstaPic lives.
**Strongly recommended on a public server:** without a relay, some players get the sound of a
call but no picture (4G box, CGNAT, strict NAT, VPN, or two players behind the same box). In an
InstaPic live they get neither, since the live's sound travels with the picture.

Two providers are built in, Cloudflare and Metered. Pick one, fill in its two fields, restart.

File: `config/turn.lua`

```lua
Config.VideoTurn = {
    provider = '',

    keyId = '',
    apiToken = '',

    meteredApp = '',
    meteredApiKey = '',
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `provider` | `string` | `''` | `'cloudflare'` or `'metered'` turns the relay on. Any other value, empty included, leaves it off |
| `keyId` | `string` | `''` | Cloudflare: the Key ID of your TURN key |
| `apiToken` | `string` | `''` | Cloudflare: the API token of that same key |
| `meteredApp` | `string` | `''` | Metered: your app name, the part before `.metered.live` (pasting the full address works too) |
| `meteredApiKey` | `string` | `''` | Metered: your API key |

The relay stays off until the provider and its two fields are filled in. Spaces around the
values are ignored.

{% hint style="danger" %}
**This file is server-side only.** It is declared in `server_scripts` in `fxmanifest.lua`, right
after `config/upload.lua`, and it must stay there. Never move it into `shared_scripts` or
`client_scripts`: a shared script is downloaded into the cache of every player who connects, and
your keys would be public. The `GetConfig` server export leaves this block out too.
{% endhint %}

## Cloudflare

1. Create a free account on [https://dash.cloudflare.com](https://dash.cloudflare.com)
2. In the dashboard, open **Realtime**, then **TURN Server**, and create a TURN key
3. Paste its Key ID into `keyId` and its API token into `apiToken` (the token is only shown
   once), and set `provider = 'cloudflare'`
4. Restart the resource

Cloudflare's free tier covers the first 1,000 GB each month, then $0.05 per GB (Cloudflare
pricing). Only calls that cannot connect directly go through the relay.

{% hint style="warning" %}
**HTTP 404** means Cloudflare does not know the Key ID: it is usually the App ID of a Realtime
SFU (Calls) app, or the account ID, instead of a TURN key. A wrong token gives **401** or **403**.
{% endhint %}

## Metered

1. Create a free account on [https://www.metered.ca](https://www.metered.ca)
2. In its dashboard, note your app name and your API key
3. Paste them into `meteredApp` and `meteredApiKey`, and set `provider = 'metered'`
4. Restart the resource

A missing key gives **HTTP 400**, an invalid one **401**.

## Your own TURN server

To use your own relay (coturn, for example) or any provider not listed here, leave `provider`
empty and add the server to `Config.VideoCall.iceServers` in
[config/main.lua](config-main.md#video-calls-facetime), with its username and password.

## Testing your values

You can check your values outside the game. If the command returns a list of servers, the
phone will work with the same values.

Cloudflare:

```bash
curl -X POST "https://rtc.live.cloudflare.com/v1/turn/keys/YOUR_KEY_ID/credentials/generate-ice-servers" -H "Authorization: Bearer YOUR_API_TOKEN" -H "Content-Type: application/json" -d "{\"ttl\": 86400}"
```

Metered:

```bash
curl "https://YOUR_APP.metered.live/api/v1/turn/credentials?apiKey=YOUR_API_KEY"
```

## What the server console says

`<Provider>` is Cloudflare or Metered.

| Line | Meaning |
| ---- | ------- |
| `[nash_phone] video: <Provider> TURN relay ready.` | The relay works |
| `[nash_phone] video: <Provider> TURN request failed (HTTP <code>), check ... in config/turn.lua. Video calls fall back to STUN only.` | The provider refused the request or could not be reached. Printed once per series of failures; the server tries again on its own, from one minute up to one hour later |
| `[nash_phone] video: <Provider> TURN relay is available again.` | The relay works again after a failure |
| `[nash_phone] video: no TURN relay configured, ... See config/turn.lua.` | Printed at startup when video calls are on and no relay is set, neither here nor in `Config.VideoCall.iceServers` |

## How it works

The keys never leave your server. The server asks the provider for relay credentials, keeps
them 4 hours and renews them when less than 2 hours remain, and only those credentials are
sent to players, with each video call and each live. The provider's servers are added after
the entries of `Config.VideoCall.iceServers`.

## Related settings

The video settings themselves (resolution, frame rate, framing, STUN servers, your own TURN
server) are in [config/main.lua](config-main.md#video-calls-facetime). The full walkthrough,
with how to read the F8 traces, is in
[Video calls and live streams](../guides/setup-video-calls.md).
