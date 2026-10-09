# Video calls and InstaPic live streams

Turn on video calls and live streams, and configure the network so players actually see each
other's image instead of only hearing them.

## Prerequisites

- Filesystem access to the `nash_phone` resource folder
- `Config.VideoCall` in `config/main.lua`
- `Config.Social.Live` in `config/social.lua`
- On a public server, a TURN relay: a free Cloudflare account (`config/turn.lua`, recommended),
  or your own TURN server. Read the next sections to see why

## What travels where: read this first, it decides the settings

During a video call the phone's camera acts as a webcam: each player sends **their own render
of the game** to the other. The same mechanism drives InstaPic live streams: the game view is
painted into a canvas, `captureStream()` turns it into a stream, and that stream travels.

| What | Route |
| ---- | ----- |
| The **handshake:** one offer, one answer, a handful of connection candidates | Through **your server**. A few kilobytes per call |
| The **image**, and the **sound of a live** | **Directly from one player to the other** when their networks allow it, otherwise **through the TURN relay**. Never through your game server |

That is what makes the feature viable on a game server: a few kilobytes per call instead of one
video stream per conversation. It is also why the network settings matter: your server has no
say in whether two players can reach each other.

{% hint style="info" %}
The server's signalling relay only forwards between **participants of a call in progress**, and
only if that call is a video call. Anything else is refused. The same guard covers lives in
progress.
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
| `longEdgePx` | `640` | 160 to 1280 | Resolution of the outgoing stream. 640 is a good compromise: above it the thumbnail gains nothing visible and bandwidth climbs fast |
| `fps` | `24` | 5 to 30 | Frames per second sent |
| `framing` | `0.35` | 0.05 to 0.95 | Where to aim inside the game image. `0.5` = centre, `0.30` = clearly left, `0.70` = right |
| `iceServers` | one Google STUN entry | - | STUN servers, and optionally your own TURN server (section 4 below) |

### Tuning `framing`

In selfie mode GTA does not put the character in the middle of the screen: they sit off to the
left. This setting says where to aim.

- If your correspondent sees you cut off on the **left** of their screen, **lower** the value.
- If they see you too far to the **right**, raise it.

It takes effect after a restart of the resource, with nothing to rebuild.

## 2. STUN and TURN

Two kinds of servers help two players connect:

| Kind | What it does |
| ---- | ------------ |
| **STUN** | Tells each player what their public address is. Free; a Google one ships in the config. It is only enough when both players' networks accept a direct connection |
| **TURN** | **Relays the image** when no direct route exists: 4G boxes, CGNAT, strict NAT, VPN, two players behind the same box |

{% hint style="warning" %}
**On a public server, set up a TURN relay.** With STUN only, every player on one of the
networks above gets the sound but no image. The rest of the call works normally: it rings, it
displays, voice goes through `pma-voice`. Only the picture is missing. This is a limit of the
network, not of the phone, and on a public server you will always have some of these players.
{% endhint %}

Only the calls that cannot connect directly use the relay. The others stay player to player.

When video calls are on and no relay is configured anywhere, the server console says so at
startup:

```
[nash_phone] video: no TURN relay configured, players behind some networks (4G box, CGNAT, strict NAT, VPN) will get sound without picture. See config/turn.lua.
```

You have two ways to add a relay: Cloudflare through `config/turn.lua` (next section), or your
own TURN server in `Config.VideoCall.iceServers` (section 4).

## 3. Cloudflare TURN (recommended)

`config/turn.lua` connects the phone to Cloudflare's TURN service. You paste one key, and the
server takes care of the rest.

1. Create a free account on [https://dash.cloudflare.com](https://dash.cloudflare.com).
2. In the dashboard, open **Realtime**, then **TURN**, and create a key.
3. Copy the key's **Key ID** and its **API token**.
4. Open `config/turn.lua`, set `provider = 'cloudflare'` and paste both values:

    ```lua
    Config.VideoTurn = {
        provider = 'cloudflare',
        keyId = 'your Key ID',
        apiToken = 'your API token',
    }
    ```

5. Restart the resource.

A moment after startup, the server console prints one of these lines:

| Line | Meaning |
| ---- | ------- |
| `[nash_phone] video: Cloudflare TURN relay ready.` (green) | The relay works. Nothing else to do |
| `[nash_phone] video: Cloudflare TURN request failed (HTTP <code>), check keyId and apiToken in config/turn.lua. Video calls fall back to STUN only.` (red) | Cloudflare refused the request, or could not be reached. Check that both values come from the same key and were pasted whole |

After a failure, calls keep working with STUN only, and the server tries again on its own,
waiting longer after each failure (from one minute up to one hour). The red line is printed
once per series of failures, and this one when the relay comes back:

```
[nash_phone] video: Cloudflare TURN relay is available again.
```

**How the key is used.** The API token never leaves your server. The server asks Cloudflare
for short-lived relay credentials (valid 4 hours, renewed when less than 2 hours remain) and
only those reach the players, inside the data of a video call or a live. A player who connects
without calling gets none.

{% hint style="danger" %}
**`config/turn.lua` is server-side only.** It is declared in `server_scripts` in
`fxmanifest.lua`, and it must stay there. Every shared script is downloaded into the cache of
each connecting player: moving this file into `shared_scripts` would hand your API token to all
of them. For the same reason, never paste the token into `config/main.lua`.
{% endhint %}

{% hint style="info" %}
**Cost.** Cloudflare's free tier covers the first **1,000 GB each month** (shared with their
other Realtime service), then **$0.05 per GB**. Only the calls and lives that go through the
relay count. Check Cloudflare's pricing page for current numbers.
{% endhint %}

## 4. Your own TURN server

If you prefer coturn, or another provider that gives fixed credentials, add the server to
`Config.VideoCall.iceServers` in `config/main.lua`. Uncomment the block and paste your
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

{% hint style="warning" %}
`config/main.lua` is a shared script, and this list is sent to every player when they connect:
fixed credentials written here can be read by anyone on your server, who could then use your
relay. Use credentials dedicated to the phone. `config/turn.lua` does not have this problem.
{% endhint %}

An unreadable entry is dropped on its own rather than invalidating the whole block, and if the
list ends up empty the public STUN is put back: without any server, two players would never
find each other and video would fail everywhere over one missing comma.

Both options can be combined: the Cloudflare servers are added after the entries of
`Config.VideoCall.iceServers`.

## 5. How many people fit in a call

Four, guests included. The image travels as a **mesh**, everyone connected to everyone: at four
participants each player holds three connections and sends three copies of their own stream.
Beyond that you would need a video mixing server, that is to say another machine to rent.

## 6. On the call screen

- The other player's picture appears once the connection is up and a first frame has arrived.
  Until then, their contact picture stays on screen.
- If the connection does not come up within 15 seconds, fails, or drops for more than 5
  seconds, the phone rebuilds it once.
- After a second failure, the call screen shows **"No picture: a direct connection could not be
  made. The call continues with audio."**, and the player's F8 console prints this line, once
  per call, even with `debug = false`:

    ```
    [nash_phone] video: no route to the other player (a TURN relay is needed, see config/turn.lua)
    ```

- **Minimizing** a video call stops your camera: the other player sees a black picture until
  you bring the call back. The connection stays up.
- **Closing the phone** stops the picture. It comes back when the player opens the phone again.

## 7. InstaPic live streams

A live broadcasts the host's actual game view to whoever is watching, with the microphone of the
host and of the guests on screen, over exactly the same route: picture and sound never pass
through the game server, and they use the same TURN relay as calls. The relay credentials come
with the live when it starts or when a viewer joins.

```lua
-- config/social.lua
Config.Social = {
    -- ...
    Live = {
        enabled = true,
        maxViewers = 8,
        audience = 'followers',
        microphone = true,
    },
}
```

| Key | Default | Accepted range | Effect |
| --- | ------- | -------------- | ------ |
| `enabled` | `true` | - | `false`: the button stays visible but the server refuses to start, and the app shows a clear refusal instead of a dead button |
| `maxViewers` | `8` | 1 to 24 | Maximum viewers per live |
| `audience` | `'followers'` | `'everyone'`, `'followers'`, `'none'` | Who is **notified** when a live starts. It is a server setting: the player does not choose it at broadcast time |
| `microphone` | `true` | `true`, `false` | `true`: viewers hear the host and the guests on screen. `false`: silent lives, with no mic button |

`'everyone'` means everyone who has an InstaPic account and is connected: not every player on
the server. Notifying someone who does not have the app tells them nothing and hands them a
notification they cannot open.

`audience` never changes who can **watch**: whatever its value, only accounts that follow the
host can join a live. An older config may still say `defaultAudience`: it is read when
`audience` is absent, and when both are filled with different values, `audience` wins and the
console says so at startup.

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

Behaviours worth knowing:

- **A live is never persisted.** There is no SQL table, deliberately: a live only means anything
  while the broadcaster is connected. An interrupted live is a finished live.
- **The list never enumerates accounts.** A player only sees lives from accounts they already
  follow, plus their own. Returning "every live on the server" would hand out a directory of
  connected players, which the whole social layer is built to avoid.
- On screen, the host plus up to **three guests** can be shown at once: the panel is tall and
  narrow, and beyond that nothing is readable.
- **One live start per minute** per player: a second start within 60 seconds is refused.
- **The sound is the microphone only.** The game's own audio is never sent: the embedded browser
  never receives GTA's audio mix. Unlike a call, a live's sound does not go through `pma-voice`:
  it travels with the picture, so a viewer with no route to the host gets neither. A viewer's
  microphone is never captured.
- **Mic button.** The host and the guests can mute and unmute from the button next to the camera
  flip button. When the microphone is refused, missing or has not answered after 5 seconds, the
  live goes on with picture only, and the button shows a red crossed-out mic that asks again
  when clicked.
- **During a phone call**, the host and the guests stop sending picture and sound, and a viewer
  stops hearing the live, until they hang up.

{% hint style="warning" %}
`Config.VideoCall.enabled = false` also removes the image and the sound from lives: the host
never opens the game camera or the microphone. The live can still be started if
`Config.Social.Live.enabled` is `true`, but nobody will see or hear anything. Turn both off if
you want the feature gone.
{% endhint %}

The "X is live" notification stays **inside the phone**. It never draws over the game screen,
unlike an incoming call.

## How to know it works

1. Check the server console: with `config/turn.lua` filled, you want the green
   `Cloudflare TURN relay ready.` line.
2. Two players, both connected, both with the phone open. Place a call from **VideoCall**, or use
   the video button of a contact in Phone or Contacts, or of a ChatApp conversation.
3. Both should see the other's game view in the call screen. Voice comes from `pma-voice` and is
   independent: a silent call is a `pma-voice` problem, not a video one.
4. If the sound works and the image does not, set `debug = true`, restart, and try again. The
   whole sequence is traced in the player's F8 console under `[nash-phone][video]`. Players are
   named by their place in the call (`p1` for the caller, `p2` for the person called, then
   guests), not by phone number:

    ```
    [nash-phone][video] offrant vers p2 appel call_12 serveurs ICE 3
    [nash-phone][video] p2 : essai 1
    [nash-phone][video] offre envoyee
    [nash-phone][video] reponse recue
    [nash-phone][video] IMAGE RECUE
    ```

    The number after `serveurs ICE` counts the servers in use. With the shipped STUN entry and
    no relay, it reads `1`: the TURN servers did not arrive.

5. A failed attempt is traced like this:

    ```
    [nash-phone][video] ECHEC : aucune connexion apres 15 s - 12 candidats envoyes, 9 recus.
    [nash-phone][video] Sans relais TURN, c est le cas normal derriere un CGNAT. Voir config/turn.lua.
    ```

    Read the two counts:

    - **Candidates received above 0:** both players exchanged their addresses, but no route
      between them works. This is the case a TURN relay fixes. If you already have one, check
      the relay line in the server console and the `serveurs ICE` count.
    - **0 received:** the other player's messages never arrived. Their phone was closed or not
      on the call, or the server refused to forward (see the next step). A TURN relay does not
      change this.

6. `RELAIS REFUSE` means the server refused to forward a handshake message. `disabled` means
   `Config.VideoCall.enabled` is `false`; `too_big` means the payload exceeded the 16 KB the
   relay accepts; `too_many` means the player sent too many messages in a short time.

7. For lives: start one from InstaPic and open the phone on a second account **that follows the
   first**. The live appears; on an account that does not follow, it must not. The viewer
   should also hear the host. With `debug = true`, the host's F8 console says what happened to
   the microphone, under the same `[nash-phone][video]` prefix:

    | Line | Meaning |
    | ---- | ------- |
    | `micro capte` | Microphone captured, the live has sound |
    | `micro trop lent, live lance sans son` | No answer after 5 seconds, the live started with picture only |
    | `micro indisponible, live sans son` | Microphone refused or missing |
    | `micro arrive en retard, flux rebranche` | A late microphone was added, connections rebuilt |
    | `nouvel essai du micro` | The player clicked the red mic button to try again |

{% hint style="warning" %}
Put `debug` back to `false` afterwards. The trace is deliberately verbose and prints on every
step of every call.
{% endhint %}
