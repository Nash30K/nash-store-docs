# config/maps.lua

Everything the Maps app shows: world bounds, travel estimation, categories and the list of
places.

File: `config/maps.lua`, loaded as a shared script.

Places are written in **game coordinates**, the ones you read standing on the spot. The phone
converts them to a position on the map tiles on its own, computes the real distance from the
player and the travel time. The whole payload is built client-side from this shared file, with
no server round trip.

{% hint style="info" %}
To read a position: stand where you want the place, type `/phonepos` in chat, and the block to
paste below is printed in your own client console (F8). See [Reading a
position](#reading-a-position).
{% endhint %}

## The map itself

The map is not stored in the resource. The Maps app, the Snapz map and the location bubbles in
Messages load Rockstar's own GTA V map tiles on the fly, the ones behind the Social Club map
(`s.rsg.sc`). Players switch between three styles with the layers button above the zoom
controls: **satellite**, **game map** and **paper map**. The choice is saved with the player's
settings.

{% hint style="warning" %}
**The map needs internet access.** Each player's game downloads the tiles when a map opens, then
keeps them in cache. If Rockstar's servers cannot be reached, the map stays a plain colour, while
places, the player's position and the GPS keep working. None of the three styles shows street
names.
{% endhint %}

## Config.Map.Bounds

Used to convert game coordinates into a 0-100% position on the map. It serves the places of the
Maps app, the Snapz map **and** location sharing in Messages.

```lua
Config.Map.Bounds = {
    minX = -4149.5,
    maxX = 4889.2,
    minY = -5140.5,
    maxY = 8417.6,
}
```

{% hint style="warning" %}
**Do not change these values.** They describe the grid of Rockstar's map tiles, which is not
square: the map is one and a half times taller than it is wide, so `maxY - minY` must stay equal
to 1.5 × `maxX - minX` (shipped: 9038.7 × 13558.1). Any other value moves every place, and the
player's own position, away from where it really is.
{% endhint %}

{% hint style="info" %}
**These numbers are measured, not estimated.** They come from matching the coastline of
Rockstar's tiles against a map calibrated in game, then checking seven landmarks: Maze Bank
Tower, Legion Square, the La Mesa Los Santos Customs, the central hospital, the Fort Zancudo
control tower, the Mount Chiliad summit and Los Santos International Airport. Each one lands on
its building, within 25 metres.
{% endhint %}

If the block is missing, or degenerate (`minX == maxX`, or `minY == maxY`), the phone falls
back to the same values internally, so a deleted block does not produce a wrong map.

With `Config.Debug = true` in [config/main.lua](config-main.md), `/phoneconfig` prints the
computed width and height and says whether they respect the proportion of the tiles, and
`/phonecheck` lists wrong bounds among the things preventing the phone from working properly.

## Config.Map.Travel

The distance and time shown on each place are computed from the player's real position. As the
crow flies is always shorter than the road, so `roadFactor` corrects the gap.

```lua
Config.Map.Travel = {
    roadFactor = 1.35,
    speedKmh = 50.0,
    refreshSeconds = 5,
}
```

| Setting | Default | Meaning |
| ------- | ------- | ------- |
| `roadFactor` | `1.35` | `1.0` = straight line. `1.35` = road estimate. A value of `0` or less falls back to `1.35` |
| `speedKmh` | `50.0` | Average speed used to estimate the minutes. A value of `0` or less falls back to `50.0` |
| `refreshSeconds` | `5` | How often the list is rebuilt, only while Maps is open |

Those two guards are not decoration: a speed of zero would produce an infinite time, which
FiveM's JSON encoder refuses, and the app would stay empty with no message at all.

## Config.Map.Categories

Each place belongs to a category, which gives it its default colour and icon. Add, remove or
rename freely.

```lua
Config.Map.Categories = {
    { id = 'place',    icon = 'location', color = 'green' },
    { id = 'hospital', icon = 'hospital', color = 'red',    chip = true },
    { id = 'police',   icon = 'police',   color = 'blue',   chip = true },
    { id = 'garage',   icon = 'garage',   color = 'orange', chip = true },
    { id = 'bank',     icon = 'bank',     color = 'indigo', chip = true },
    { id = 'airport',  icon = 'airport',  color = 'teal' },
}
```

| Field | Description |
| ----- | ----------- |
| `id` | Identifier used by places, in their `category` field |
| `label` | Displayed name. Leave it empty for the six original categories: the phone already translates those into French and English |
| `icon` | One of the icon names below |
| `color` | `red` `orange` `yellow` `green` `teal` `blue` `indigo` `purple` `pink` `gray`, or a direct colour code such as `'#ff2d55'` |
| `chip` | `true` adds a filter shortcut under the search bar |

## Config.Map.Points

### Required fields

| Field | Description |
| ----- | ----------- |
| `id` | Unique identifier, no spaces |
| `name` | Displayed name |
| `category` | One of the `id` values declared in `Config.Map.Categories` |
| `coords` | Position in the game, read with `/phonepos`. Accepts `vec3(x, y, z)`, `{ x = , y = }` or `{ x, y }` |

### Optional fields

| Field | Description |
| ----- | ----------- |
| `subtitle` | Grey line under the name in the result list. Shown **as written**, in whatever language you type it. Leave it out and the phone uses the category name instead, which it does translate |
| `address` | Address shown on the place card. Falls back to `subtitle` |
| `icon` | Overrides the category icon |
| `color` | Overrides the category colour |
| `phone` | Number shown on the card. The Call button dials it, so put a real service number there if you have one |
| `open24` | `true` = always open, displayed as 24/7 |
| `openHour` / `closeHour` | Opening hours in **game time** (0 to 23). The place then shows as Open or Closed depending on the current hour |

`open24` wins over the hours. A range that crosses midnight works: `openHour = 20`,
`closeHour = 4`. A range where `openHour == closeHour`, or where either is missing, means
always open.

The game hour is read **once per refresh** and applied to the whole list, so two places in the
same list can never answer on two different hours.

### Available icons

```
location  hospital  police   garage    bank     airport   shop     restaurant
bar       fuel      home     work      gym      hotel     boat     helicopter
car       tools     cash     ticket    beach    school    star     flag
bag
```

### Example

```lua
Config.Map.Points = {
    {
        id = 'pillbox',
        name = 'Pillbox Hill Medical',
        category = 'hospital',
        address = 'Strawberry Avenue, Los Santos',
        coords = vec3(307.7, -595.0, 43.3),
        phone = '555-0911',
        open24 = true,
    },
    {
        id = 'lscustoms',
        name = 'Los Santos Customs',
        category = 'garage',
        address = 'Little Bighorn Avenue, Los Santos',
        coords = vec3(-337.2, -136.5, 39.0),
        phone = '555-0464',
        openHour = 8,
        closeHour = 22,
    },
}
```

{% hint style="info" %}
The six places shipped in the file are **examples**, placed on landmarks of the stock game map.
Replace them with the places of your own map.
{% endhint %}

## Reading a position

`/phonepos` is a client command. Stand where you want the place and run it: the phone prints
the map percentage and a ready-to-paste block into your own client console (F8), already filled
with the current street and zone.

```
[nash-phone] carte : 52.6 % en largeur, 61.4 % en hauteur
[nash-phone] bloc à coller dans config/maps.lua :
    {
        id = 'monlieu',
        name = 'Mon lieu',
        category = 'place',
        address = 'Strawberry Avenue, Los Santos',
        coords = vec3(307.7, -595.0, 43.3),
        open24 = true,
    },
```

Rename `id` and `name`, pick a `category`, and paste it into `Config.Map.Points`.

The block never carries a `subtitle`: without one the phone shows the category name, which it
translates itself, and the address already carries the street and the zone. Apostrophes in
street names such as O'Neil Way are escaped, so the pasted block still loads.

`/phonepos` does not require `Config.Debug`, unlike the `/phone*` test commands: it only prints
to the console of the player who ran it and writes nothing.
