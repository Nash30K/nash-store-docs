# config/upload.lua

Where the photos and videos taken with the phone are hosted. **This file must be filled in for
the Camera to work.**

File: `config/upload.lua`

```lua
Config.Upload = {
    provider = 'fivemanage',
    apiKey = '',
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `provider` | `string` | `'fivemanage'` | The only supported host. Any other value disables uploads |
| `apiKey` | `string` | - | Your Fivemanage media API token |

{% hint style="danger" %}
**This file is server-side only.** It is declared in `server_scripts` in `fxmanifest.lua` and it
must stay there. Never move it into `shared_scripts` or `client_scripts`: a shared script is
downloaded into the cache of every player who connects and can be read with any text editor.
Your API key would be public, and anyone could spend your quota.

The phone does not need it on the player's side. The client only asks the server a yes/no
question: "is a host configured?": and the files travel either through the server or through a
single-use upload address.
{% endhint %}

{% hint style="warning" %}
**The key that comes with the script is not yours.** If your copy of `config/upload.lua` arrives
with an `apiKey` already filled in, it is a trial key shared by every copy of the product:
replace it with your own before going live, or your server's photos are hosted on somebody
else's account and count against somebody else's bandwidth bill. If it arrives empty, the Camera
stays disabled until you fill it in.
{% endhint %}

## Getting a key

Two minutes, free account, no credit card.

1. Create an account on [https://fivemanage.com](https://fivemanage.com)
2. Dashboard → **API Tokens** → generate a token for **media**
3. Paste it into `apiKey`
4. Restart the resource

## What happens with no key

The server prints a red block in the console at startup:

```
[nash_phone] Config.Upload is NOT configured -> the CAMERA IS DISABLED.
[nash_phone] The phone only stores a LINK to each picture, so an image host
[nash_phone] is required. Set this in config/upload.lua:
[nash_phone]    provider = 'fivemanage'  + apiKey = '<Fivemanage API key>'
```

In game, the Camera refuses to take a picture and warns the player, and the app says so
**before** recording a video rather than letting them film for thirty seconds for nothing.

## Why a host is needed at all

The phone does not store images: it stores a **link** to an image. Without a host there is
nothing to link to.

## Why not a Discord webhook

Because it no longer works. Since December 2023 Discord signs its CDN addresses and they expire
after 24 hours:

```
.../photo.png?ex=<expiry>&is=<issued>&hm=<signature>
```

Past that delay the address returns 404, and stripping the parameters changes nothing. Pictures
post to the channel, display perfectly, and the whole camera roll turns into a grid of broken
thumbnails the next day. That failure is invisible at install time, which is why the mode was
removed rather than left in with a warning.

Fivemanage returns a permanent address, with no parameter and no signature:

```
https://r2.fivemanage.com/image/gLxgYgvTZe99.png
```

## How the upload works

All of it is handled by `server/services/upload.lua`. There are two paths, and one of them is
no longer used by anything:

| Path | Used for | Note |
|---|---|---|
| **Presigned** | Everything: videos filmed by the interface, and stills posted by `screenshot-basic` | The server asks Fivemanage for a temporary address and returns **only that address**. The key stays on the server |
| **Relay** | No caller left | The file comes up in base64 and the server posts it. Kept for a host that cannot presign, and capped at 200 KB accordingly |

The relay was the photo path until it turned out that a reliable network event cannot carry an
image: past 393,216 bytes FiveM **kicks the player**.

Both paths share one rate limit, because they are two halves of the same upload: **12 uploads
per minute per player**. Removing the key from the client stops it being read, not being used -
a modified client could otherwise ask for presigned addresses in a loop and drain the owner's
quota without even having a file to send.

## Related settings

Photo and video resolution, video length, bitrate and microphone are configured in
[config/main.lua](config-main.md#camera).
