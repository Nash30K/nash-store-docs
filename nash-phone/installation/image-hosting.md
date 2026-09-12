# Image Hosting

The phone does not store images. It stores a **link** to an image. So it needs somewhere to
put the file, and that is the one thing you must configure by hand before going live.

Everything happens in `config/upload.lua`.

## What stops working without it

| Feature | Without a host |
|---|---|
| Camera photos | Refuses to shoot, warns the player, red line in the console |
| Camera videos | Same |
| Snapz and ChatApp snaps | No in-game capture can be saved |
| Photos, add by link | The **+** button of the Albums tab does not appear: a pasted link would have nowhere to be re-hosted |
| Camera roll, profile pictures, social posts | Nothing new to display: they all read links produced by the features above |

`/phonecheck` lists a missing host as a non-blocking issue, and `/phonedeps` tells you whether
the problem is the host or `screenshot-basic`.

## Getting a token

Fivemanage is the only supported provider. A free account is enough and does not ask for a
card.

1. Create an account on [fivemanage.com](https://fivemanage.com).
2. Dashboard → **API Tokens** → generate a token for **media**.
3. Paste it into `apiKey` in `config/upload.lua`.
4. Restart the resource.

```lua
Config.Upload = {
    provider = 'fivemanage', -- the only supported host
    apiKey = 'your-token-here',
}
```

The file ships with an empty `apiKey`. As long as it stays empty, `Config.Upload.provider`
being set changes nothing: the server answers "no host configured" and the camera refuses
before even hiding the interface, so the player does not get a pointless flash.

## `config/upload.lua` is a server script

{% hint style="danger" %}
`config/upload.lua` is declared in `server_scripts` and **must stay there**. Never move it to
`shared_scripts` or `client_scripts`.

Every shared script is downloaded into the cache of every player who connects, and can be read
with any file browser. Your API token would become public, and anyone could burn your quota on
your account.
{% endhint %}

This is why it is the only config file loaded outside the `shared_scripts` block in
`fxmanifest.lua`, and why the block that loads the others carries a comment saying so.

The client never needs the token. It asks the server one question ("is a host configured?")
and gets back a single boolean:

| Callback | Returns |
|---|---|
| `camera:upload:available` | `{ ok = true, configured = <boolean> }` |

That answer is cached on the client for the session: the hosting configuration does not change
mid-game, and the Camera app asks on every open.

## How a file actually travels

| Path | Who posts the file | Used by |
|---|---|---|
| **Presigned** | The client, straight to the host | Camera stills (posted by `screenshot-basic` from its own page), videos, and pictures added by link in the Photos app |
| **Relay** | The server, from a base64 payload | Fallback only. Kept for a host that cannot presign, and for the browser preview |

The presigned path exists because the image must not travel through the FiveM network at all.
A reliable network event drops its packet past **393 216 bytes**, and the server then kicks the
player (`Reliable network event size overflow`). A 1920×1080 JPEG at quality 0.85 weighs about
330 KB, which is roughly 440 KB once base64-encoded, over the limit, on every single photo.
Splitting it into chunks does not help: the limiter counts bytes, not events.

So the server asks Fivemanage for a single-use address and returns **only that address**. The
token never leaves the server.

The relay is capped at **200 KB** of binary. It refuses oversized payloads before decoding
them, both on the encoded length and on the decoded one.

## Rate limit

Both callbacks share one counter, because they are two halves of the same upload:

| Setting | Value |
|---|---|
| Window | 60 seconds |
| Points per window | 24 |
| Cost of a presign request | 2 |
| Cost of a relay upload | 2 |

That is **12 uploads per player per minute**. The limit is checked before the round trip to
Fivemanage, since it is that round trip, and the quota it spends on your account, that a
modified client would try to run in a loop.

Adding a photo by link draws on the same counter and the same account. An import asks for up to
**two** presigned addresses, one for the picture and one for its thumbnail, so a player can add
about six pictures a minute. A link that is already on Fivemanage is uploaded again like any
other, so that the copy lives on your account and follows your size settings.

## Error codes

Errors reach the interface as short codes; the useful detail goes to the server console.

| Code | Meaning |
|---|---|
| `no_upload` | No host configured, or `apiKey` empty |
| `no_screenshot` | `screenshot-basic` is not started |
| `invalid_type` | File type outside `image`, `audio`, `video` |
| `too_many` | Rate limit hit |
| `too_large` | Relay payload over 200 KB |
| `presign_bad_response` | The host answered something that is not JSON |
| `presign_refused` | The host refused the request: usually a wrong or revoked token |
| `presign_no_url` | The host answered `ok` with an empty address |
| `no_file`, `decode_failed` | Empty or unreadable base64 payload on the relay path |
| `http_<status>`, `no_url` | The host rejected the relay upload, or returned no address |

Adding a photo by link does not show these codes: the player gets a readable message instead
("This link has expired", "This image is too large"...). The full list, with the cause of each,
is in [Common Errors](../common-errors.md#adding-a-photo-by-link-fails).

## Why not a Discord webhook

Because it stopped working. Since December 2023 Discord signs its CDN addresses and they
expire after 24 hours:

```
.../photo.png?ex=<expiry>&is=<issued>&hm=<signature>
```

After that the address answers 404, and stripping the parameters changes nothing. Photos post
fine, display perfectly, and the whole camera roll turns into a grid of broken thumbnails the
next day. That failure is invisible at install time, which is why the mode was removed rather
than left in with a warning.

Fivemanage returns an address with no parameter, no signature and no expiry of its own. How long
the file behind it lives is up to your account: see the next section.

Players can still bring a picture posted on Discord into their phone, with the **+** button of
Photos. The phone downloads it while the link still works and re-hosts it on Fivemanage; the
Discord link itself is never stored. See
[Adding photos by link](../config/config-main.md#adding-photos-by-link).

## How long links last

Every address the phone stores, and every link a player copies out of Photos, stays valid as
long as your Fivemanage account keeps the file. That is a setting of **your Fivemanage
account**, not of the phone:

| Retention on your Fivemanage account | What happens to the files |
|---|---|
| None | Kept until you delete them yourself |
| 7, 30, 90, 180 or 365 days (set per media type) | Deleted for good once they are older than that |

The phone does not ask Fivemanage to exempt its files, so a retention policy applies to camera
photos, videos and pictures added by link alike. Once a file is gone, the camera roll shows a
crossed-out picture in its place ("Image Unavailable" in the full-screen viewer), and a link
pasted elsewhere stops working. Leave retention off if your players expect their camera roll to
last.

A copied link is public: anyone who has it can open the file. Deleting a photo from the phone
removes the row from the camera roll, not the file from Fivemanage.

## Quality and weight

The size of what you upload is set in `Config.Camera` (`config/main.lua`), not here.

| Key | Default | Note |
|---|---|---|
| `Photo.LongEdgePx` | `1920` | Long edge of the still. `1920` gives 1440×1920 in portrait |
| `Photo.Quality` | `0.8` | WebP quality, `0.0` → `1.0` |
| `Recording.MaxDurationSeconds` | `30` | Automatic stop |
| `Recording.Fps` | `30` | Captured frames per second |
| `Recording.Bitrate` | `1200000` | Bits per second, about 150 KB per second of video |
| `Recording.LongEdgePx` | `720` | Accepted range 240 to 1080. **The heaviest setting for in-game performance**: video is encoded live while GTA runs, and the cost follows the pixel count. Going to 1080 multiplies it by 2.25 without a sharper image, since the bitrate stays the same |
| `Recording.Microphone` | `true` | Record the filming player's voice. `false` removes the button from the app |

Game audio cannot be recorded: the embedded browser never receives the GTA audio mix.

Pictures added by link in Photos are re-encoded with the same `Photo.LongEdgePx` and
`Photo.Quality`, so an imported picture weighs about as much as a camera photo, whatever the
size of the original.
