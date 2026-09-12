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

## Config.Map.Bounds

Used to convert game coordinates into a 0-100% position on the map tiles. It serves both the
places of the Maps app **and** location sharing in Messages.

```lua
Config.Map.Bounds = {
    minX = -5718.2,
    maxX = 6776.7,
    minY = -4083.1,
    maxY = 8411.8,
}
```

{% hint style="warning" %}
**The two spans must stay equal.** The map tiles are square: one tile covers as many metres
wide as it does tall. If `maxX - minX` differs from `maxY - minY`, the projection stretches one
axis and places drift the further they sit from the centre. The shipped values are
12494.9 x 12494.9.
{% endhint %}

{% hint style="info" %}
**These numbers are measured, not estimated.** They come from six landmarks located on the
tiles themselves (Maze Bank Tower, Legion Square, the La Mesa Los Santos Customs, the central
hospital, the Fort Zancudo control tower, the Mount Chiliad summit) matched against their game
coordinates. Los Santos International Airport was deliberately kept **out** of that
calculation and used as a control: it lands on its terminal. The remaining error is 10 to 25
metres in the city.

Change them only if your server actually uses a different map. Nudging them "to see" will
undo a calibration you cannot easily redo by eye.
{% endhint %}

To fine-tune: stand somewhere very recognisable, run `/phonepos`, add the place to the list
below and open Maps. If the dot lands off target, shift **both** bounds of the same axis
together (for example `minX` and `maxX` by +200 each) so you do not reintroduce a stretch.

If the block is missing, or degenerate (`minX == maxX`, or `minY == maxY`), the phone falls
back to the same measured values internally, so a deleted block no longer produces a wrong
map. Keep the block anyway: it is the only place you can adapt the phone to a custom map.

With `Config.Debug = true` in [config/main.lua](config-main.md), `/phoneconfig` prints the
computed width and height and says whether they are square, and `/phonecheck` lists non-square
bounds among the things preventing the phone from working properly.

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
