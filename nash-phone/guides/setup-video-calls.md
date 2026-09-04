# Video calls and InstaPic live streams

Turn on video calls and live streams, and configure the network so players actually see each
other's image instead of only hearing them.

## Prerequisites

- Filesystem access to the `nash_phone` resource folder
- `Config.VideoCall` in `config/main.lua`
- `Config.Social.Live` in `config/social.lua`
- Optionally, a TURN server: rented or self-hosted (coturn). Read the next section before
  deciding whether you need one

## What travels where: read this first, it decides the settings

During a video call the phone's camera acts as a webcam: each player sends **their own render
of the game** to the other. The same mechanism drives InstaPic live streams: the game view is
painted into a canvas, `captureStream()` turns it into a stream, and that stream travels.

| What | Route |
| ---- | ----- |
| The **handshake:** one offer, one answer, a handful of connection candidates | Through **your server**. A few kilobytes per call |
| The **image** | **Directly from one player to the other.** It never passes through your server |

That is what makes the feature viable on a game server: a few kilobytes per call instead of one
video stream per conversation. It is also why the network settings matter: your server has no
say in whether two players can reach each other.

{% hint style="info" %}
The relay only forwards between the **two actual participants of a call in progress**, and only
if that call is a video call. Anything else is refused. The same guard covers lives, addressed
by username instead of phone number.
{% endhint %}

## 1. The video call block

```lua
-- config/main.lua
Config.VideoCall = {
    enabled = true,
    debug = false,
    longEdgePx = 640,
    fps = 24,
    framing = 0.35,
    iceServers = {
        { urls = 'stun:stun.l.google.com:19302' },
    },
}
```

| Key | Default | Accepted range | Effect |
| --- | ------- | -------------- | ------ |
| `enabled` | `true` | - | `false` = video calls stay audio, with no image |
| `debug` | `false` | - | Writes the whole course of the video call into the player's F8 console |
| `longEdgePx` | `640` | 160 – 1280 | Resolution of the outgoing stream. 640 is a good compromise: above it the thumbnail gains nothing visible and bandwidth climbs fast |
| `fps` | `24` | 5 – 30 | Frames per second sent |
| `framing` | `0.35` | 0.05 – 0.95 | Where to aim inside the game image. `0.5` = centre, `0.30` = clearly left, `0.70` = right |

### Tuning `framing`

In selfie mode GTA does not put the character in the middle of the screen: they sit off to the
left. This setting says where to aim.

- If your correspondent sees you too far **right** on their screen (so you are cut off on the
  left), **lower** the value.
- If they see you too far left, raise it.

It takes effect on the next call, with nothing to rebuild.

## 2. ICE servers, STUN and TURN

`iceServers` is the list of servers that help two players find each other.

| Kind | What it does |
| ---- | ------------ |
| **STUN** | Tells each player what their public address is. Free; the Google one shipped in the config is enough in most cases |
| **TURN** | **Relays the image** when a direct connection is impossible: players behind CGNAT, some 4G boxes, corporate networks |

{% hint style="warning" %}
**Without a TURN server, those players get the sound but no image.** The rest of the call works
normally: it rings, it displays, voice goes through `pma-voice`. Only the picture is missing.
This is a limit of the network, not of the phone.
{% endhint %}

A TURN server is rented or self-hosted (coturn). Uncomment the block and paste your
credentials. **Keep UDP + TCP + TLS:** that is what gets through the most firewalls.

```lua
Config.VideoCall = {
    enabled = true,
    debug = false,
    longEdgePx = 640,
    fps = 24,
    framing = 0.35,
    iceServers = {
        { urls = 'stun:stun.l.google.com:19302' },
        {
            urls = {
                'turn:turn.example.com:3478?transport=udp',
                'turn:turn.example.com:3478?transport=tcp',
                'turns:turn.example.com:5349?transport=tcp',
            },
            username = 'user',
            credential = 'password',
        },
    },
}
```

An unreadable entry is dropped on its own rather than invalidating the whole block, and if the
list ends up empty the public STUN is put back: without any server, two players would never
find each other and video would fail everywhere over one missing comma.

## 3. How many people fit in a call

Four, guests included. The image travels as a **mesh**, everyone connected to everyone: at four
participants each player holds three connections and sends three copies of their own stream.
Beyond that you would need a video mixing server, that is to say another machine to rent.

## 4. InstaPic live streams

A live broadcasts the host's actual game view to whoever is watching, over exactly the same
route: the image never passes through the server.

```lua
-- config/social.lua
Config.Social = {
    -- ...
    Live = {
        enabled = true,
        maxViewers = 8,
        defaultAudience = 'followers',
    },
}
```

| Key | Default | Accepted range | Effect |
| --- | ------- | -------------- | ------ |
| `enabled` | `true` | - | `false`: the button stays visible but the server refuses to start, and the app shows a clear refusal instead of a dead button |
| `maxViewers` | `8` | 1 – 24 | Maximum viewers per live |
| `defaultAudience` | `'followers'` | `'everyone'`, `'followers'`, `'none'` | Value offered on the launch screen; the player changes it at broadcast time |

`'everyone'` means everyone who has an InstaPic account and is connected: not every player on
the server. Notifying someone who does not have the app tells them nothing and hands them a
notification they cannot open.

{% hint style="danger" %}
**`maxViewers` is not a comfort setting, it is a physical limit.** With no media server, the
broadcaster sends a **separate** stream to each viewer: their upstream bandwidth and their CPU
climb proportionally. It is the same reason a call is capped at four.

At 8, a player on fibre holds up without trouble. Past a dozen the broadcast degrades **for
everybody**, including viewers who were already connected, and the broadcaster loses frames in
their own game. Raising the number does not fit more people in: it spreads the same capacity
over more viewers.

The live stays open when the ceiling is reached: newcomers get a readable refusal, they do not
cut anyone off.
{% endhint %}

Two behaviours worth knowing:

- **A live is never persisted.** There is no SQL table, deliberately: a live only means anything
  while the broadcaster is connected. An interrupted live is a finished live.
- **The list never enumerates accounts.** A player only sees lives from accounts they already
  follow, plus their own. Returning "every live on the server" would hand out a directory of
  connected players, which the whole social layer is built to avoid.
- On screen, the host plus up to **three guests** can be shown at once: the panel is tall and
  narrow, and beyond that nothing is readable.

{% hint style="warning" %}
`Config.VideoCall.enabled = false` also removes the image from lives: the host never opens the
game camera. The live can still be started if `Config.Social.Live.enabled` is `true`, but
nobody will see anything. Turn both off if you want the feature gone.
{% endhint %}

The "X is live" notification stays **inside the phone**. It never draws over the game screen,
unlike an incoming call.

## How to know it works

1. Two players, both connected, both with the phone. Place a call from **VideoCall**, or use the
   video button in a ChatApp conversation.
2. Both should see the other's game view in the call screen. Voice comes from `pma-voice` and is
   independent: a silent call is a `pma-voice` problem, not a video one.
3. If the sound works and the image does not, set `debug = true`, restart, and try again. The
   whole sequence is traced in the player's F8 console under `[nash-phone][video]`:

    ```
    [nash-phone][video] offrant vers 555-0143 appel c12 serveurs ICE 1
    [nash-phone][video] offre envoyee
    [nash-phone][video] reponse recue
    [nash-phone][video] IMAGE RECUE
    ```

4. The failure you are looking for is explicit:

    ```
    [nash-phone][video] ECHEC : aucune route entre les deux joueurs. 12 candidats envoyes, 0 recus.
    [nash-phone][video] Sans serveur TURN, c est le cas normal derriere un CGNAT. Voir Config.VideoCall.
    ```

    Candidates sent but none received is the CGNAT signature. Add a TURN server.

5. `RELAIS REFUSE` means the server refused to forward a handshake message. `disabled` means
   `Config.VideoCall.enabled` is `false`; `too_big` means the payload exceeded the 16 KB the
   relay accepts.

6. For lives: start one from InstaPic and open the phone on a second account **that follows the
   first**. The live appears; on an account that does not follow, it must not.

{% hint style="warning" %}
Put `debug` back to `false` afterwards. The trace is deliberately verbose and prints on every
step of every call.
{% endhint %}
