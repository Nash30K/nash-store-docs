# First Launch

## Start order in `server.cfg`

```cfg
# Dependencies first
ensure oxmysql
ensure ox_lib
ensure ox_inventory        # or your inventory

# Framework
ensure es_extended         # or qb-core / qbx_core

# Optional
ensure pma-voice
ensure screenshot-basic
ensure xsound

# The phone
ensure nash-phoneprop
ensure nash_phone
```

`oxmysql` and `ox_lib` must start before `nash_phone`: they are hard dependencies, along with
OneSync. `nash-phoneprop` must start before it too, or the model held in hand falls back to a
GTA prop.

## Server console at startup

The phone reports the state of the install in the server console, at every start. There is
nothing to enable and no command to run: start the resource and read the
lines that begin with `[nash_phone]`. Most of them are in French, and are quoted below exactly
as they appear.

A healthy start prints these lines (the example is ESX with `ox_inventory`):

```
[nash_phone] framework détecté : esx.
[nash_phone] inventaire détecté : ox.
[nash_phone] schéma DB prêt (31 tables).
[nash_phone] Langue « fr » chargee : … texte(s) d ecran, … message(s).
[nash_phone] Langue « en » chargee : … texte(s) d ecran, … message(s).
```

The other checks stay silent when everything is fine. The item checks run **15 seconds**
after the framework is detected, to let the inventory finish loading its item list, so wait
that long before concluding.

| Line | Meaning |
|---|---|
| `[nash_phone] framework détecté : esx.` | The framework bridge resolved. Replace `esx` with `qbcore` / `qbox` |
| `[nash_phone] framework : esx (imposé par la configuration).` | Same, but forced through `Config.Framework` |
| `[nash_phone] Config.Framework = "…" : aucun pont de ce nom. Ponts disponibles : …` | `Config.Framework` names a bridge that does not exist. Use one of the names listed, or `auto`. See [Requirements](requirements.md#framework-and-inventory-selection) |
| `[nash_phone] inventaire détecté : ox.` | The inventory bridge resolved (`ox`, `qs`, `qb`, `framework`) |
| `[nash_phone] inventaire : ox (imposé par la configuration).` | Same, but forced through `Config.Inventory`. A custom inventory prints `inventaire : pont personnalisé.` |
| `[nash_phone] Config.Inventory = "…" : inconnu. Attendu : auto, ox, qs, qb, framework, custom.` | `Config.Inventory` holds a value the phone does not know |
| `[nash_phone] schéma DB prêt (31 tables).` | The database is ready. Extra text appears when columns or indexes were added. If this line never appears, read the database error printed above it. See [Database](database.md) |
| `[nash_phone] Langue « fr » chargee : … texte(s) d ecran, … message(s).` | One line per language file loaded from `locales/`. See [Add a language](../guides/add-language.md) |
| `[nash_phone] locales/….json est introuvable.` or `… est illisible (JSON invalide). Langue ignoree.` | A language file is missing or broken, and that language is skipped |
| `[nash_phone] AUCUN framework détecté pour l'instant. Le téléphone ne pourra identifier personne.` | No player can be identified yet. Start your framework before `nash_phone`. The phone asks again on every call, so it starts working as soon as the core is up, but the usable item is only registered if the framework appears within 60 seconds |
| `[nash_phone] Config.UseItem = true but the item "phone" DOES NOT EXIST` | Red block: nobody can open the phone. See [Inventory items](inventory-items.md) |
| `[nash_phone] Config.Earbuds is on but the item "phone_earbuds" does not exist…` | Yellow: nobody can pair earbuds, nothing else changes |
| `[nash_phone] Config.Upload is NOT configured -> the CAMERA IS DISABLED.` | Red block: no image host. See [Image hosting](image-hosting.md) |
| `[nash_phone] video: no TURN relay configured, …` | Yellow: some players will get video calls with sound but no picture. See [config/turn.lua](../config/config-turn.md) |
| `[nash_phone] video: Cloudflare TURN relay ready.` | Green: the TURN relay of `config/turn.lua` works. A red `Cloudflare TURN request failed (HTTP …)` line means the request failed. See [config/turn.lua](../config/config-turn.md) |
| `[nash_phone] config/apps.lua : attention aux consequences ci-dessous.` | You disabled `appstore`, `settings`, `phone` or `messages`. See [Choose which apps are installed](../guides/customize-apps.md) |
| `[nash_phone] index … non créé : …` | An index could not be added. Nothing breaks, some lists get slower |

One more line appears in the **player's F8 console**, not in the server console:
`[nash_phone] modèle « custom_phone_prop » introuvable, repli sur…` means `nash-phoneprop` is
missing or not started. See [Streamed prop](streamed-prop.md).

The resource prints nothing sensitive at boot.

## In-game checks

The console tells you what is missing. These steps confirm the rest works. Text messages
and calls need a second player.

1. **Open the phone.** Press `F1` (`Config.OpenKey`, remappable by each player in the FiveM
   key settings) or type `/phone` (`Config.OpenCommand`). With `Config.UseItem = true` you need
   the `phone` item in your pockets. If nothing opens, look for the red
   `DOES NOT EXIST` block in the server console.
2. **The setup wizard runs.** A new character goes through the first-open sequence: hello,
   language, country, appearance, byCloud account and identity (only with
   `Config.Setup.cloudAccount.enabled`), privacy, Face ID (only with
   `Config.Setup.faceId.enabled`), passcode. It appears once per character (and again on a new
   handset, when your inventory tells handsets apart), and the current step is saved from the
   very first screen, so disconnecting mid-way resumes where you left off.
   `Config.Setup.enabled = false` skips it entirely and opens straight on the lock screen.
3. **A number is assigned.** It is generated the first time the character is loaded (or at the
   first opening of the phone) and never changes hands. Check it in Settings, or from another
   resource (the server is the only one that writes this state bag):

   ```lua
   local number = Player(source).state.phoneNumber
   ```

4. **Send a text message** to another player's number, then check the two rows land:

   ```sql
   SELECT owner, number FROM nash_phone_phones;
   SELECT sender, receiver, mtype FROM nash_phone_messages ORDER BY id DESC LIMIT 5;
   ```

5. **Place a call.** Without `pma-voice` the call still rings, connects and is logged. It is
   simply silent. That is expected.
6. **Take a photo.** This is the check that fails most often: it needs a token in
   `config/upload.lua`. If the red `Config.Upload is NOT configured` block appeared at startup,
   this step cannot pass. `screenshot-basic` is only used as a fallback when the phone cannot
   read the game view, and for the automatic profile pictures of Snapz and ChatApp.
7. **Look at the phone in third person.** The model should be the NASH one, not a stock GTA
   handset. If it is not, type `/nashprop`: it prints in the F8 console the state of
   `nash-phoneprop` and whether the model loads.

The full list of commands the phone registers is in [Commands](../developer-api/commands.md).

## The three most common failures

**Photos are not saved.** Almost always, the token in `config/upload.lua` was never filled: the
server console then prints the red `Config.Upload is NOT configured` block at every start. If
the token is there, take a photo and read the server console. `presign fivemanage: refus (…)`
means Fivemanage refused the request, usually a wrong or revoked token. A yellow block starting
with `Photo/video refusee` means the domain serving your files is missing from
`Config.Gallery.Import.AllowedHosts` (`config/main.lua`). Also make sure `screenshot-basic` is
started, since the Camera falls back on it when it cannot read the game view.

**Calls are silent.** `pma-voice` is not started. The call screen still works. That is expected,
not a bug.

**The phone is two-tone in hand.** `assets/prop/` did not arrive complete during the copy: the
chassis changes colour but the camera block keeps the previous colour. See
[Streamed prop](streamed-prop.md).

## If something still does not work

Head to [Common Errors](../common-errors.md).
