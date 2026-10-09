# Commands

NASH Phone registers the command players use every day, a cursor key binding, a camera key binding, two client-side helpers anyone can run (`/phonepos` and `/nashprop`), and one server console command.

## Player commands

<details>

<summary>/phone</summary>

Opens and closes the phone.

```
/phone
```

**Permissions:** none.

**Command name:** `Config.OpenCommand` in `config/main.lua`, default `phone`.

**Key binding:** `Config.OpenKey`, default `F1`, registered through `RegisterKeyMapping` so each player can rebind it in *Settings → Key Bindings*. Setting `Config.OpenKey = ''` registers no binding and leaves only the command.

**Behavior**

- While a call or an InstaPic live stream is in progress, the command does not put the phone away: hanging up is done with the on-screen buttons.
- Closing is instant and local. Opening goes through the server (`nash_phone:requestOpen`) so the inventory item can be checked.
- If the phone has been confiscated by another script through the `ToggleDisabled` export, the player gets a notification instead of an open phone: the key does not just look broken.
- A player without the item gets an `ox_lib` error notification ("You don't have a phone.", in their language), whatever `Config.NotifyStyle` is set to.
- While the phone is open, `Esc` and `Backspace` also put it away, except during a call, a live stream, while the camera is in use, or while a system sheet is open on screen.
- The key binding label is written in the server language (`Config.Locale`).

{% hint style="warning" %}
With `Config.UseItem = true` (the default) the item named in `Config.ItemName` must exist in your inventory, or **nobody** can open the phone: the command, the key and the `ToggleOpen` / `OpenApp` client exports all check it server-side. The resource prints a red console warning 15 seconds after start when the item is missing.
{% endhint %}

</details>

<details>

<summary>/phonepos</summary>

Prints the player's current position, formatted as a ready-to-paste `config/maps.lua` entry, into the **client** console (F8).

```
/phonepos
```

**Permissions:** none. It only reveals the caller's own coordinates.

**Output:** one line with the position on the map in percent, one header line, then a block (the console lines and the placeholder `id` and `name` are written in French):

```lua
{
    id = 'monlieu',
    name = 'Mon lieu',
    category = 'place',
    address = 'Vespucci Boulevard, Del Perro',
    coords = vec3(-1234.5, 456.7, 22.3),
    open24 = true,
},
```

Street names containing an apostrophe are escaped, so the block pastes without breaking the config file. No `subtitle` is emitted on purpose: without it the phone shows the translated category name.

</details>

<details>

<summary>/nashprop</summary>

Diagnoses the 3D phone prop and its colour shell, in the **client** console (F8). This is the first thing to ask for on a "my prop does not show up" ticket.

```
/nashprop
```

**Permissions:** none.

**What it reports**

- State of the `nash-phoneprop` resource
- Requested model name and its `joaat` hash
- Whether the game actually loads the model, and after how long (5 s ceiling)
- The configured fallback model and whether it is currently loaded
- The shell / skin settings, and for each colour whether its back image is found

Its output is written in French.

If the model does not load, the command names the two usual causes: `nash-phoneprop` not ensured **before** `nash_phone`, or a `stream/` folder missing its `.ydr` / `.ytd` / `.ytyp`.

</details>

<details>

<summary>nashphone_curseur</summary>

Key binding, default `Left Ctrl` (`Config.LookKey`). While the phone is open, it hides the cursor and gives the mouse back to the game camera, so the player can look around and keep walking. Pressing it again brings the cursor back. Any screen that needs the cursor (an incoming call, the end of a capture) brings it back on its own.

While the Camera app is active (also during a video call or a live), the same key shows or hides the cursor over the camera screen. The on-screen hints display the key the player actually bound. Crouching is blocked while the phone is open. The label is written in the server language (`Config.Locale`) in the FiveM key bindings screen.

</details>

<details>

<summary>nashcam_photo</summary>

Camera app key binding. It is registered as a command so FiveM can expose it in *Settings → Key Bindings*, but it only does something while the camera is active.

| Command | Default key | Effect |
|---|---|---|
| `nashcam_photo` | `Enter` | Takes the picture. Ignored during a call and while a capture is already running |

The label goes through the locale system and are written in the server language (`Config.Locale`) in the FiveM key bindings screen.

</details>

## Server console commands

<details>

<summary>nashphone_cleanup</summary>

Finds gallery rows whose media was hosted on Discord and removes them. Those links have expired and are unrecoverable: the pictures no longer exist, only the database rows do.

```
nashphone_cleanup            # counts, deletes nothing
nashphone_cleanup confirm    # deletes
```

**Permissions:** **server console only** (`source == 0`). The command is registered as restricted, and even a player who was granted its ACE is refused by the code: physical access to the console is the right barrier here.

**Behavior:** without `confirm` it reports how many rows are affected and how many players are concerned, and changes nothing. With `confirm` it deletes them, and turns every image SMS whose address contains `discordapp.` back into text, even a recent one, so the conversation keeps a trace instead of an empty bubble. When no gallery row matches, it stops there and does not touch the SMS. Its console output is written in French.

**What it matches:** gallery rows whose address contains `discordapp.`, whatever wrote them.

- **Photos added by link are never affected**, even when the player pasted a Discord link: Photos re-hosts the picture on Fivemanage before saving it, so the stored address never points at Discord.
- The `SaveToGallery` export only accepts addresses on a host listed in `Config.Gallery.Import.AllowedHosts`. A Discord address can only reach the gallery that way if that list was widened to include Discord, which the config advises against; such rows are matched like any other.

</details>

## Adding your own command

The phone uses plain `RegisterCommand` and exposes no permission helper. To keep a command of your own for staff, register it as restricted and grant its ACE:

```lua
RegisterCommand('phonenumberof', function(source, args)
    local target = tonumber(args[1])
    if not target then return end

    local number = exports.nash_phone:GetEquippedPhoneNumber(target)
    print(('%s has number %s'):format(target, number))
end, true) -- restricted: needs the command.phonenumberof ACE, the server console always passes
```

```
add_ace group.admin command.phonenumberof allow
```

See [Server Exports](server-exports.md) for the full list.
