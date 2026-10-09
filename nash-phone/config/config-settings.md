# config/settings.lua

The name of your server's city, as it is displayed **inside** the phone.

File: `config/settings.lua`

```lua
Config.City = {
    name = 'Los Santos',
    region = 'San Andreas',
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | `'Los Santos'` | City name. Used everywhere, on its own and inside sentences |
| `region` | `string` | `'San Andreas'` | State or region, shown next to the city in the Compass ("Los Santos, San Andreas"). Leave empty to show only the city |

## Where it shows up

The lock-screen weather widget, the home-screen widget, the Weather app, Clock, Compass, the
App Store, the social networks (an InstaPic post is tagged with the city name), and the country
list of the first-run setup when `config/setup.lua` offers none.

The phone ships set to Los Santos, GTA V's base map. If your server runs a custom map (Chicago,
Detroit, a European city), replace the name here: there is nowhere else to edit, nothing is
hard-coded in the screens. The value is validated and cached at startup, then sent to the
interface at bootstrap.

An empty or non-string `name` falls back to `Los Santos`: a screen with a hole in it is never
an acceptable outcome. `region` may legitimately be empty: the Compass then shows the city
alone, with no orphan comma.

## Set this before opening your server

{% hint style="warning" %}
What was saved with the old name keeps it: an InstaPic post stores the city name it was
published with, and a player who finished the first-run setup keeps the country they picked.
Every other screen follows the new name after a restart of the resource.
{% endhint %}

## What this setting does not change

Other GTA-universe names stay in place: the Clock app's time zones (Liberty City, Vice City),
the Stocks app's companies (Maze Bank, Lifeinvader) and the Maps app's street names. Those are
nods, not city names: changing them would mean rewriting the content of those apps.

The map's points of interest are configured separately, in [config/maps.lua](config-maps.md).
