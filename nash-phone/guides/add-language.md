# Add a language

Translate the whole phone into a language of your choice by dropping one JSON file into
`locales/` and changing a single config line: nothing to compile.

## Prerequisites

- Filesystem access to the `nash_phone` resource folder
- A text editor that saves in **UTF-8**
- Roughly a full day of work: a complete language is about **2,300 strings**

{% hint style="info" %}
**Why the language files are JSON read at runtime.** You receive the interface already built
(`web/build/`), without its sources. A language written inside the interface code could
therefore only ever be added by us. Languages instead live in `locales/<code>.json` and are
read from disk when the resource starts (`shared/uilocales.lua`), which is exactly what lets
you add one without rebuilding anything.
{% endhint %}

## What is in a language file

The `locales/` folder holds one file per language, and nothing else.

```
locales/fr.json      shipped
locales/en.json      shipped
locales/zh.json      yours
```

Every file carries two blocks, and that is all there is to know:

| Block | What it holds | Count |
| ----- | ------------- | ----- |
| `dict` | Everything displayed **on the phone screen** | ~2,300 strings |
| `lua` | Messages displayed **outside the phone**: notifications, warnings, key-mapping labels | 44 strings |

## 1. Duplicate a shipped file

```
locales/en.json   ->   locales/zh.json
```

Start from English or French, whichever you translate more comfortably. The file is already
complete, so no key can slip past you.

Then change the two values at the top:

```json
"code": "zh",
"label": "中文",
```

`label` is the name shown in Settings, and it is written **in the language itself**: a
Chinese player looks for 中文, not for "Chinese".

## 2. Translate the screen (`dict`)

Translate the **right-hand side**. Never touch the key on the left: it is what ties the
string to the screen that displays it.

```json
"settings.general": "General",
"settings.general": "通用"
```

**Anything inside curly braces is not translated.** `{city}`, `{n}` and `{name}` are values
the phone inserts. Move them wherever your language requires, but keep them intact.

```json
"weather.credit": "{city} Weather · Simulated data",
"health.stepsGoal": "{n} steps out of {goal}"
```

**The `localeTag` keys are not sentences.** `en-US` is a code that drives the formatting of
dates, times and numbers. Left as-is, your phone will display "3:45 PM" and "1,234.5" in the
middle of an otherwise fully translated interface. Replace it with your own language's tag:
`zh-CN`, `es-ES`, `pt-BR`. There are eleven of them:

```
birdby.localeTag      calculator.localeTag   calendar.localeTag
health.localeTag      messages.localeTag     notes.localeTag
phone.localeTag       shell.lock.localeTag   snapz.localeTag
stocks.localeTag      wallet.localeTag
```

The file must stay in **UTF-8**. Every alphabet works.

## 3. Translate the out-of-phone messages (`lua`)

Same file, further down. The same rules apply, with one difference: the placeholders are
written `%s` and `%.1f` instead of `{name}`. Keep them, in the same order.

```json
"missed_call": "Missed call from %s",
"missed_call": "来自 %s 的未接来电"
```

The nine strings that carry placeholders:

```
location_shared    📍 Location shared (%.1f, %.1f)
message_from       Message from %s
missed_call        Missed call from %s
number_assigned    Your number: %s
services_new_body  %s needs your help.
services_taken_body %s is responding to your request.
transfer_done      You sent $%s to %s.
transfer_recv      You received $%s from %s.
verify_code        Your %s verification code is %s. Do not share it with anyone.
```

## 4. Enable it

One line:

```lua
-- config/main.lua
Config.Locale = 'zh'
```

Restart the resource.

`Config.Locale` sets two things at once: the language of the notifications pushed outside the
phone, and the starting language of a player who has never touched the setting. Each player
keeps the final say through **Settings → General → Language**, and their choice is remembered.

## 5. (Optional) Let players choose

If you want several languages **offered** in Settings → General → Language, list the extra
ones:

```lua
-- config/main.lua
Config.ExtraLocales = { 'zh', 'es' }
```

French and English are always in that list. `Config.ExtraLocales` is only about *offering* a
choice: to simply move the whole server to another language, `Config.Locale` is enough.

## How to know it works

At startup, the server prints one green line per loaded language:

```
[nash_phone] Langue « zh » chargee : 2336 texte(s) d ecran, 44 message(s).
```

The two counts are the real number of keys read from your file. A count far below the shipped
one means your file lost a block while you were editing it.

If the language is missing, the console says so, in red:

```
[nash_phone] locales/zh.json est introuvable. Langue « zh » ignoree.
[nash_phone] locales/zh.json est illisible (JSON invalide). Langue ignoree.
```

Three causes, in order of frequency:

1. The filename does not match the code: `locales/zh.json` for `'zh'`.
2. The JSON is invalid, most often a trailing comma on the last line.
3. You wanted to **offer** the language to players without adding it to `Config.ExtraLocales`.

In game, open Settings: the footer, the SIM card page and the app names should all be in your
language. Then lock the phone: the lock screen date is the fastest way to check that you
replaced the `localeTag` values.

## What happens when a string is missing

Nothing breaks. An untranslated string falls back to **English**, then to French, and only as
a last resort displays its raw key (`settings.general`). A key showing up on screen is a
signal, not a crash: it leads you straight to the line to complete.

You can therefore enable a partial translation and finish it later.

French and English are also **embedded in the compiled interface**. A deleted or malformed
`fr.json` does not empty the phone: it simply falls back to ours.

## Editing our French or our English

Same gesture, without duplicating anything: open `locales/fr.json` and change the sentence
that does not fit your roleplay. Both files are read at startup like any other.

{% hint style="warning" %}
`fr.json` and `en.json` are **shipped files**, so they are replaced on the next phone update.
Note your changes somewhere so you can reapply them. A language *you* added is never touched
- we do not ship a `zh.json`.
{% endhint %}

## After a phone update

An update sometimes adds strings. Those fall back to English until you pick them up: nothing
breaks, and you catch up at your own pace. Compare the new `locales/en.json` with yours to
spot the missing keys.

## What stays in its original language

**Proper nouns**: your server's name, the city name, the application names. They are set
elsewhere, in `config/main.lua` (`Config.Brand`) and `config/settings.lua` (`Config.City`).

`Config.Brand.site.title` is the one branding value that is written **per language:** the key
is the language code:

```lua
-- config/main.lua
Config.Brand = {
    name = 'NASH',
    studio = 'Nash',
    site = {
        label = 'NASH Store',
        url = 'nash-store.com',
        title = {
            fr = 'NASH Store : Scripts FiveM',
            en = 'NASH Store: FiveM Scripts',
            zh = 'NASH Store : FiveM 脚本',   -- one line per language you added
        },
    },
}
```

A language with no line here simply displays `label`.

## Language codes

Use the usual ISO code: `es` Spanish, `de` German, `it` Italian, `pt` Portuguese, `pt-br`
Brazilian Portuguese, `nl` Dutch, `pl` Polish, `ru` Russian, `tr` Turkish, `ar` Arabic, `zh`
Chinese, `ja` Japanese, `ko` Korean.

The code has no special meaning to the phone: it only ties `Config.Locale` to the filename. It
just has to be the same on both sides.
