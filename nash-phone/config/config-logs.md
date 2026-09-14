# config/logs.lua

Discord and Fivemanage logs of what players do with the phone: posts, stories, live streams,
photos, messages, transfers, reports. Two kinds of logs can run side by side: **admin** logs for
your staff, with the player's full identity, and **public** logs that feed roleplay on your
Discord, with the content only.

File: `config/logs.lua`

```lua
Config.Logs = {
    enabled = true,

    admin = {
        service = 'discord',
        webhook = '',
        fivemanageKey = '',
        dataset = 'default',
    },

    public = {
        webhook = '',
    },

    embed = true,
    image = true,
    footer = 'NASH Phone',

    events = {
        instapic_post = { admin = true, public = false, color = '#E1306C' },
        -- one line per event, see below
    },
}
```

{% hint style="danger" %}
**This file is server-side only.** It is declared in `server_scripts` in `fxmanifest.lua`, right
after `config/upload.lua`, and it must stay there. A webhook link is a password: anyone who reads
it can post anything in your channels. Every shared script is downloaded into the cache of each
connecting player, so moving this file into `shared_scripts` would hand your webhooks to all of
them.
{% endhint %}

Nothing is sent until you fill in a webhook or a Fivemanage token: the shipped file is inert.

## General settings

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | `false` turns every log off |
| `admin.service` | `string` | `'discord'` | Where admin logs go: `'discord'`, `'fivemanage'` or `'both'` |
| `admin.webhook` | `string` | `''` | Default Discord webhook for admin logs |
| `admin.fivemanageKey` | `string` | `''` | Fivemanage token used for logs. Empty: the key of [config/upload.lua](config-upload.md) is tried |
| `admin.dataset` | `string` | `'default'` | Fivemanage dataset the logs are written to |
| `public.webhook` | `string` | `''` | Default Discord webhook for public logs |
| `embed` | `boolean` | `true` | `false` sends a plain text message instead of an embed |
| `image` | `boolean` | `true` | Shows the picture attached to the action, when there is one |
| `footer` | `string` | `'NASH Phone'` | Text at the bottom of each log. `''` for none |

## Admin logs and public logs

| Shown | Admin | Public |
|---|---|---|
| App, title, time | Yes | Yes |
| Author: `@username` and profile picture, phone number or mail address | Yes | Yes |
| Text and picture of the post | Yes | Yes |
| Place, service, video link | Yes | Yes |
| Recipient, amount, memo, subject, reported account, reason, duration | Yes | No |
| Character name and server ID | Yes | No |
| Phone number, framework identifier, Discord mention | Yes | No |

The Discord mention is a clickable `<@id>` read from the player's identifiers: it points your
staff to the account, and it never pings anyone.

{% hint style="warning" %}
**Public logs exist only for posts, stories, live streams and service requests.** Text messages,
private messages, mails, calls, photos, transfers, account creations and reports can never be
sent publicly, even if you set `public` on them: the setting is ignored for those events.

A public service request shows the requester's character name, the one the employees already see
in game. It is meant for a job channel, such as a dispatch channel reserved for your police.
{% endhint %}

## Events

| Event | Fires when | Public possible | Shipped |
|---|---|---|---|
| `instapic_post` | An InstaPic post is published | Yes | Admin |
| `instapic_story` | An InstaPic story is published | Yes | Admin |
| `instapic_live` | An InstaPic live stream starts | Yes | Admin |
| `birdby_post` | A BirdBy post is published | Yes | Admin |
| `ticktok_post` | A TickTok video is published | Yes | Admin |
| `snapz_post` | A Snapz post is published | Yes | Admin |
| `snapz_story` | A Snapz story is published | Yes | Admin |
| `chatapp_story` | A ChatApp status is published | Yes | Admin |
| `service_request` | A citizen sends a written request in Services | Yes | Admin |
| `camera_photo` | A photo is taken with the Camera | No | Admin |
| `camera_video` | A video is recorded with the Camera | No | Admin |
| `gallery_import` | A photo is added from a link in Photos | No | Admin |
| `bank_transfer` | A transfer is sent from Wallet | No | Admin |
| `social_account` | An account is created in a social app | No | Admin |
| `cindr_report` | A Cindr profile is reported | No | Admin |
| `sms` | A text message is sent | No | Off |
| `social_message` | A private message is sent in a social app | No | Off |
| `mail` | A mail is sent | No | Off |
| `call` | A call starts, voice or video, Services calls included | No | Off |

The four private communication events ship turned off: turning them on means your staff reads
players' conversations, which is a decision for your server rules.

An event that is missing from `events` is never logged.

## Per-event settings

```lua
events = {
    birdby_post = {
        admin = true,
        public = 'https://discord.com/api/webhooks/000000000000000000/xxxxxxxx',
        color = '#1D9BF0',
        embed = true,
        image = true,
        name = 'BirdBy',
        avatar = 'https://example.com/birdby.png',
    },
}
```

| Key | Type | Description |
|---|---|---|
| `admin` | `true`, link or `false` | `true` uses `admin.webhook`, a link sends to that channel, `false` sends no admin log |
| `public` | `true`, link or `false` | `true` uses `public.webhook`, a link sends to that channel, `false` sends no public log |
| `color` | `string` | Colour of the embed bar, as `'#RRGGBB'`. Default: the app's colour |
| `embed` | `boolean` | Overrides the general `embed` for this event |
| `image` | `boolean` | Overrides the general `image` for this event |
| `name` | `string` | Name shown as the sender of the message. Default: the app's name |
| `avatar` | `string` | `https` link to the sender's picture. Default: the picture set on the webhook |

With `admin.service = 'fivemanage'`, `admin = true` or a link both send the admin log to
Fivemanage only. With `'both'`, it goes to the Discord channel and to Fivemanage.

## Embed or plain text

**Embed** (`embed = true`): an author line with the `@username` and its profile picture, the title
("New post"), the text, the picture, the details as fields, the footer and the time, with the
colour bar of the app.

**Plain text** (`embed = false`): a first line in bold with the app and the title, then the
author and the text, one line per detail, and the picture link on the last line, which Discord
previews on its own.

A video is never used as the picture of a log: it appears as a **Video** link instead, since
Discord cannot play a video inside an embed.

## Setting up Discord

1. In Discord, open the channel settings: **Integrations** → **Webhooks** → **New Webhook**, then
   **Copy Webhook URL**.
2. Paste the link into `admin.webhook` for your staff channel, into `public.webhook`, or directly
   on an event for a dedicated channel.
3. `restart nash_phone`.
4. With `Config.Debug = true` in [config/main.lua](config-main.md), run `/phonelogtest`: every
   active event sends one sample log. `/phonelogtest birdby_post` sends only that one.

One public channel per app, with every action still logged for your staff:

```lua
Config.Logs.admin.webhook = 'https://discord.com/api/webhooks/.../staff'

Config.Logs.events.instapic_post = { admin = true, public = 'https://discord.com/api/webhooks/.../instapic', color = '#E1306C' }
Config.Logs.events.birdby_post   = { admin = true, public = 'https://discord.com/api/webhooks/.../birdby', color = '#1D9BF0' }
Config.Logs.events.ticktok_post  = { admin = true, public = 'https://discord.com/api/webhooks/.../ticktok', color = '#FE2C55' }
```

Write these lines at the end of `config/logs.lua`, or edit the matching lines of the `events`
table directly.

## Setting up Fivemanage

1. In your Fivemanage dashboard, open **Logs** and note the dataset you want to write to.
2. Create an API token with access to logs, and paste it into `admin.fivemanageKey`.
3. Set `admin.service` to `'fivemanage'` or `'both'`, and `admin.dataset` to your dataset.
4. `restart nash_phone`, then `/phonelogtest`.

Logs are sent in batches, every three seconds. Each entry carries:

| Field | Content |
|---|---|
| `level` | `info`, or `warn` for a Cindr report |
| `message` | App, title and author, for example `BirdBy : New post @vivewagner` |
| `resource` | The resource name, `nash_phone` |
| `metadata.event` | The event name, such as `birdby_post` |
| `metadata.app`, `author`, `text`, `image`, `link`, `place`, `service` | The content of the action |
| `metadata.to`, `subject`, `amount`, `memo`, `target`, `reason`, `duration` | The details of the action |
| `metadata.player` | `name`, `serverId`, `identifier`, `number`, `discord` |

{% hint style="info" %}
The key of `config/upload.lua` is a media token. If Fivemanage refuses the logs with it (HTTP 401
or 403 in the console), create a token that covers logs and set it in `admin.fivemanageKey`.
{% endhint %}

## Language

Titles and field names follow the server language, `Config.Locale` in
[config/main.lua](config-main.md). They are read from the `lua` block of `locales/<code>.json`
(keys starting with `log_`), so [adding a language](../guides/add-language.md) translates the logs
as well.

## Limits and safety

- **No pings.** A player who writes `@everyone`, `@here` or a role mention in a post notifies
  nobody: every log is sent with mentions disabled.
- **Discord rate limits.** Each webhook has its own queue, with a short pause between messages,
  and the phone waits whenever Discord asks it to slow down. Up to 200 messages can wait per
  webhook; past that, new logs for that webhook are dropped until the queue drains.
- **Text length.** Texts are cut to Discord's limits (title 256 characters, text 4000, field 1000,
  plain message 2000).
- **Test commands.** The fake content of `/phonefakepost`, `/phonefakestory` or `/phonefakemsg`
  does not produce logs: only real player actions and `/phonelogtest` do.

## Troubleshooting

`[nash_phone] Logs : Discord a refusé un envoi (HTTP 404). Vérifiez le lien du webhook dans config/logs.lua.`

The webhook was deleted, or the link was pasted incomplete. HTTP 401 means the token part of the
link is wrong, HTTP 400 that Discord rejected the message itself. The line is printed at most once
every five minutes per webhook.

`[nash_phone] Logs : Fivemanage a refusé un envoi (HTTP 401). Vérifiez fivemanageKey et dataset dans config/logs.lua.`

The token is wrong or does not cover logs. Check `admin.fivemanageKey` and `admin.dataset`.

**`/phonelogtest` answers that no log is active.** Every event has either no link to send to, or
`admin` and `public` set to `false`. Fill in `admin.webhook`, `public.webhook` or a Fivemanage token.

## Related pages

- [config/upload.lua](config-upload.md): the Fivemanage media key
- [Commands](../developer-api/commands.md): `/phonelogtest`
- [Server Events](../developer-api/server-events.md): the events to write your own logging
