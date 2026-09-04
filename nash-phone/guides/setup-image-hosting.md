# Set up image hosting

Give the phone somewhere to store pictures, so the camera, the gallery, profile pictures and
every social post that carries an image can actually save something.

{% hint style="danger" %}
This is **mandatory**. While `config/upload.lua` is empty, the camera refuses to take pictures,
warns the player on screen, and the server prints a red banner at every start.
{% endhint %}

## Prerequisites

- Filesystem access to the `nash_phone` resource folder
- `screenshot-basic` started on the server: it is what captures the game view
- A free [fivemanage.com](https://fivemanage.com) account. No credit card

## Why an external host is needed

The phone does **not** store images. It stores a **link** to the image. Something has to serve
that link, and the FiveM CEF browser cannot host anything itself.

## 1. Get your key

1. Create an account on [https://fivemanage.com](https://fivemanage.com).
2. Dashboard → **API Tokens** → generate a token for **media**.
3. Copy it.

Takes about two minutes.

## 2. Paste it into `config/upload.lua`

```lua
-- config/upload.lua
Config.Upload = {
    provider = 'fivemanage', -- the only supported host
    apiKey = 'PASTE_YOUR_KEY_HERE',
}
```

{% hint style="warning" %}
The file ships with an example key that is **not yours**. Left in place, that account hosts
your server's pictures and receives the bandwidth bill. Replace it before going live.
{% endhint %}

## 3. Leave the file where it is

`config/upload.lua` is declared in `server_scripts` in `fxmanifest.lua`, and it must **stay
there**.

{% hint style="danger" %}
**Never move it to `shared_scripts` or `client_scripts`.** A shared script is downloaded into
the cache of every player who connects and can be read with any text editor. Your API key
would be public, and anyone could burn your quota.

The phone does not need it on the player's side: the client only ever asks the server "is a
host configured?" and gets back a yes/no. Files travel either through the server or through a
single-use upload address.
{% endhint %}

## 4. Restart

```
restart nash_phone
```

## How to know it works

1. The red startup banner is gone. While the key is missing you get:

    ```
    ========================================================================
    [nash_phone] Config.Upload is NOT configured -> the CAMERA IS DISABLED.
    [nash_phone] The phone only stores a LINK to each picture, so an image host
    [nash_phone] is required. Set this in config/upload.lua:
    [nash_phone]    provider = 'fivemanage'  + apiKey = '<Fivemanage API key>'
    [nash_phone] Free account, no credit card: https://fivemanage.com
    ========================================================================
    ```

2. Open the Camera in game and take a picture. It appears in the gallery, and the thumbnail
   loads.
3. Reconnect and open the gallery again. The picture is still there: that is what tells you
   the link is permanent and not a session artefact.
4. With `Config.Debug = true` in `config/main.lua`:

    ```
    /phonedeps    -> tells you whether screenshot-basic is started
    /phoneconfig  -> Hebergeur   fivemanage, cle renseignee
    /phonecheck   -> reports "aucun hebergeur de medias" when the key is still missing
    ```

### If pictures still do not save

Two causes, and `/phonedeps` tells them apart:

- the key in `config/upload.lua` was not replaced;
- `screenshot-basic` is not started.

## Why not a Discord webhook

Because it no longer works. Since December 2023 Discord signs its CDN addresses and they
expire after 24 hours:

```
.../photo.png?ex=<expiry>&is=<issued>&hm=<signature>
```

Past that delay the address returns 404, and stripping the parameters changes nothing. The
pictures do land in the channel, display perfectly… and the entire camera roll turns into a
grid of broken thumbnails the next day. That failure is invisible at install time, which is
why the mode was removed rather than left in place behind a warning.

Fivemanage returns a permanent address, with no parameter and no signature:

```
https://r2.fivemanage.com/image/gLxgYgvTZe99.png
```

### Cleaning up media left over from a webhook install

If your gallery was built before the switch, those rows now point at dead links and cannot be
recovered: the expired address does not carry the message id that would let anyone ask for a
fresh one. A server console command removes them.

```
nashphone_cleanup            counts, deletes nothing
nashphone_cleanup confirm    deletes
```

Server console only (`source 0`); a player typing it in game is refused. Without `confirm` it
prints how many media rows are affected and how many players they belong to. With `confirm` it
deletes those gallery rows, and converts image messages pointing at the same addresses back
into text so the conversation keeps its trace instead of showing an empty bubble.

## Upload limits

Uploads are rate-limited per player, and both halves of an upload share one counter:
**12 uploads per minute per player**. The limit exists because removing the key from the
client prevents *reading* it, not *using* it: each upload request consumes your Fivemanage
quota whether or not a file follows.

## Related settings

Picture and video quality live in `config/main.lua`, not in `config/upload.lua`:

```lua
Config.Camera = {
    Photo = {
        LongEdgePx = 1920,       -- longest side of the still, in pixels
        Quality = 0.8,           -- webp quality (0.0 -> 1.0)
    },
    Recording = {
        MaxDurationSeconds = 30, -- automatic stop
        Fps = 30,
        Bitrate = 1200000,       -- ~150 KB per second
        LongEdgePx = 720,        -- accepted range: 240 to 1080
        Microphone = true,       -- record the voice of the player filming
    },
}
```

{% hint style="info" %}
`Recording.LongEdgePx` is the single setting with the biggest impact on in-game performance:
the video is encoded live while GTA V is running, and the cost follows the pixel count. Going
from 720 to 1080 multiplies it by 2.25 without a sharper image, because `Bitrate` stays the
same and gets spread over more pixels. Drop to 480 on a heavily loaded server.

Game audio cannot be recorded: the embedded browser never receives GTA's audio mix. Only the
player's microphone can be. `Microphone = false` removes the option entirely.
{% endhint %}
