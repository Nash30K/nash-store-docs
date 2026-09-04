# Add locations to the Maps app

Replace the sample points with the landmarks of your own map, so the Maps app lists the places
that actually exist on your server, with real distances and travel times.

## Prerequisites

- Filesystem access to the `nash_phone` resource folder
- Everything happens in `config/maps.lua`
- Being able to walk to each location in game to read its coordinates

Locations are given in **game coordinates:** the ones you read by standing on the spot. The
phone converts them to a position on the map tiles by itself, computes the real distance from
the player and estimates the travel time.

## 1. Read a position in game

Stand where you want the point, then type `/phonepos` in the chat. The block to paste is
printed in the F8 console, already filled in with the street and the zone:

```
[nash-phone] carte : 52.3 % en largeur, 41.8 % en hauteur
[nash-phone] bloc à coller dans config/maps.lua :
    {
        id = 'monlieu',
        name = 'Mon lieu',
        category = 'place',
        address = 'Vespucci Boulevard, Los Santos',
        coords = vec3(441.0, -982.0, 30.7),
        open24 = true,
    },
```

`/phonepos` is always available: it is not one of the debug commands and does not need
`Config.Debug`.

## 2. Paste the block into `Config.Map.Points`

Change `id` and `name`, pick a category, and you are done. Required fields:

| Field | Type | Description |
| ----- | ---- | ----------- |
| `id` | `string` | Unique identifier, no spaces |
| `name` | `string` | Displayed name |
| `category` | `string` | One of the `id` values declared in `Config.Map.Categories` |
| `coords` | `vec3` | Position in the game world, read with `/phonepos` |

Optional fields:

| Field | Type | Description |
| ----- | ---- | ----------- |
| `subtitle` | `string` | Grey line under the name in the result list. Displayed **as written, in the language you write it**. Leave it out and the phone puts the category name there instead, which it does translate |
| `address` | `string` | Address shown on the detail sheet. Falls back to `subtitle` |
| `icon` | `string` | Overrides the category icon |
| `color` | `string` | Overrides the category colour |
| `phone` | `string` | Number shown on the sheet. The Call button dials it, so put a real service number there if you have one |
| `open24` | `boolean` | `true` displays 24/7 |
| `openHour` / `closeHour` | `number` | Opening hours in **game time** (0 to 23). The place then shows as Open or Closed depending on the current hour. A schedule that crosses midnight works: `openHour = 20, closeHour = 4` |

A complete example:

```lua
-- config/maps.lua
Config.Map.Points = {
    {
        id = 'missionrow',
        name = 'Mission Row PD',
        category = 'police',
        address = 'Vespucci Boulevard, Los Santos',
        coords = vec3(441.0, -982.0, 30.7),
        phone = '555-0017',
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

## 3. Adjust the categories if needed

Each location belongs to a category, which gives it its default colour and icon.

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
| `id` | Identifier used by locations in their `category` field |
| `label` | Displayed name. Leave it empty for the six shipped categories: the phone already translates those into French and English |
| `icon` | See the icon list below |
| `color` | `red` `orange` `yellow` `green` `teal` `blue` `indigo` `purple` `pink` `gray`, or a direct colour code such as `'#ff2d55'` |
| `chip` | `true` adds a filter shortcut under the search bar |

Available icons:

```
location  hospital  police   garage    bank     airport   shop     restaurant
bar       fuel      home     work      gym      hotel     boat     helicopter
car       tools     cash     ticket    beach    school    star     flag
bag
```

## 4. Check the world bounds

`Config.Map.Bounds` converts game coordinates into a 0–100 % position on the map tiles. It is
used both by the Maps app and by location sharing in Messages.

```lua
Config.Map.Bounds = {
    minX = -5730.0,
    maxX = 6270.0,
    minY = -4000.0,
    maxY = 8000.0,
}
```

{% hint style="danger" %}
**The two spans must stay equal.** Map tiles are square: one tile covers as many metres wide
as it does tall. If `maxX - minX` differs from `maxY - minY`, the projection stretches one
axis and locations drift further off the further they are from the centre. The shipped values
are 12000 × 12000.

To fine-tune, shift **both** bounds of the same axis together (for example `minX` and `maxX`
by +200 each) so you do not reintroduce a stretch.
{% endhint %}

You only need to touch this if you run a custom map whose world extends beyond the base game.

## 5. Tune the travel estimate

The distance and time shown on each location are computed from the player's real position. A
straight line is always shorter than the road, so `roadFactor` corrects the gap.

```lua
Config.Map.Travel = {
    roadFactor = 1.35,   -- 1.0 = straight line, 1.35 = road estimate
    speedKmh = 50.0,     -- average speed used to estimate the minutes
    refreshSeconds = 5,  -- refresh rate while Maps is open
}
```

## How to know it works

1. `restart nash_phone`, then open Maps in game.
2. Your locations appear in the list, with a distance and a travel time that change as you
   move. If the distance does not move, the point is not reading your position: check that
   `coords` is a `vec3` and not a string.
3. Tap a location: the sheet shows the address, the opening state, and a Call button if you
   gave it a `phone`.
4. To verify the projection, add a location on a very recognisable landmark and look at where
   the pin lands on the tiles. If it is off, the bounds are wrong: not the coordinates.
5. With `Config.Debug = true`, `/phoneconfig` reports how many points and categories the
   server actually read, and `/phonecheck` flags non-square bounds explicitly:

    ```
    bornes de carte non carrees : les lieux de Plans tombent a cote.
    ```

{% hint style="warning" %}
**A location that is missing `id`, `name` or a readable `coords` is dropped silently:** no
error, it simply never appears in the app. If a point you added is missing, check those three
fields first.

A location whose `category` does not match any declared category id is still displayed, but
inherits nothing from it: no colour, no icon, and the raw category id is shown where the
category label would be. Declare the category first, then the points that use it.
{% endhint %}
