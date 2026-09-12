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

## Console checks

| Line | Meaning |
|---|---|
| `[nash_phone] framework détecté : esx.` | The framework bridge resolved. Replace `esx` with `qbcore` / `qbox` |
| `[nash_phone] framework : esx (imposé par la configuration).` | Same, but forced through `Config.Framework` |
| `[nash_phone] inventaire détecté : ox.` | The inventory bridge resolved (`ox`, `qb`, `framework`) |
| `[nash_phone] schéma DB prêt (27 tables).` | The database is ready. Extra text appears when columns or indexes were added. See [Database](database.md) |
| `[nash_phone] AUCUN framework détecté. Le téléphone ne pourra identifier personne.` | Fatal in practice: no player can be identified. Check that your framework is started before `nash_phone` |
| `[nash_phone] Config.UseItem = true but the item "phone" DOES NOT EXIST` | Nobody can open the phone. See [Inventory items](inventory-items.md) |
| `[nash_phone] modèle « custom_phone_prop » introuvable, repli sur…` | `nash-phoneprop` is missing or started too late |
| `[nash_phone] index … non créé : …` | An index could not be added. Nothing breaks, some lists get slower |
| `[nash_phone] COMMANDES DE TEST ACTIVEES (Config.Debug).` | `Config.Debug` is still `true`. Set it back to `false` before opening the server |

The resource prints nothing sensitive at boot.

## In-game checks

1. **Open the phone.** Press `F1` (`Config.OpenKey`, remappable by each player in the FiveM
   key settings) or type `/phone` (`Config.OpenCommand`). With `Config.UseItem = true` you need
   the `phone` item in your pockets.
2. **The setup wizard runs.** A new character goes through the first-open sequence: hello,
   language, country, appearance, byCloud account, identity, privacy, Face ID, passcode. It
   appears once per character, and the current step is saved from the very first screen, so
   disconnecting mid-way resumes where you left off. `Config.Setup.enabled = false` skips it
   entirely and opens straight on the lock screen.
3. **A number is assigned.** It is generated on the first bootstrap and never changes hands.
   Check it in Settings, or from another resource:

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
6. **Take a photo.** This is the check that fails most often: it needs both
   `screenshot-basic` started and a token in `config/upload.lua`.
7. **Look at the phone in third person.** The model should be the NASH one, not a stock GTA
   handset.

## Diagnostic commands

```lua
Config.Debug = true    -- config/main.lua, then restart the resource
```

Who can run them, once `Config.Debug` is on:

| Path | Detail |
|---|---|
| The server console | Always. Type the name **without** a slash: `phonecheck` |
| A framework group | `Config.DebugGroups`, `{ 'admin', 'superadmin' }` by default. Being an admin is enough |
| The `nash_phone.debug` ACE | Still supported, for FiveM's native permission system |

```cfg
add_ace identifier.license:YOUR_LICENSE nash_phone.debug allow
```

An empty `Config.DebugGroups` is a valid choice: it means "no group at all", and the ACE
becomes the only way in. `Config.Debug` is tested **before** any of the three: being an admin
never bypasses it.

| Command | What it answers |
|---|---|
| `/phonecheck` | The one to run before opening the server: only what **prevents** the phone from working, nothing else |
| `/phonedeps` | State of every dependency and what stops working without each one, plus a live query against the database |
| `/phoneschema` | Verifies the expected tables, columns and indexes are there |
| `/phoneconfig` | The configuration **as the server read it**: the fastest way to spot a badly written block that was silently ignored |
| `/phonediag` | Full report on one phone, app by app. From the console: `phonediag <player id>` |
| `/phonebase` | Real row count per table, all characters included |
| `/phonehelp` | Lists all 39 test commands |

{% hint style="info" %}
`/phoneconfig` is worth running once on every install. It prints the applied values (brand,
locale, mail domain, number format, map bounds) rather than what is written in the files, so
a block that failed to load shows up immediately.
{% endhint %}

## Test commands

The other 32 commands exist so you can test the phone **alone**, without a second player: they
fabricate an incoming mail, a received message, a call, a story, a transfer.

| Area | Commands |
|---|---|
| Content | `/phoneseed`, `/phonecount`, `/phonewipe`, `/phonefakebank`, `/phonefakenotif`, `/phonebattery`, `/phoneearbuds` |
| Calls, SMS, mail | `/phonefakecall`, `/phonecall`, `/phonehangup`, `/phonefakesms`, `/phonesms`, `/phoneseedsms`, `/phonefakemail`, `/phonemail`, `/phonecomms` |
| Social networks | `/phonefakemsg`, `/phonefakepost`, `/phonefakestory`, `/phonefakelike`, `/phonefakecomment`, `/phonefakefollow`, `/phonesocialnotifs`, `/phonesocialseed`, `/phonesocial`, `/phonesocialwipe`, `/phonesnapmap`, `/phonelive`, `/phonelivewho` |
| Setup wizard and Face ID | `/phonesetup`, `/phonesetupreset`, `/phonefaceid` |

Useful examples:

```
/phoneseed                      fills your phone: contacts, SMS, photos, wallet, notes, notifications
/phonefakecall                  an incoming call arrives (add "video" for a video call)
/phonefakebank 500 Paycheck     simulates an incoming transfer (a negative amount debits)
/phonesetupreset                replay the setup wizard on the next open
/phonewipe confirmer            erases ALL of your own phone data
```

{% hint style="danger" %}
Set `Config.Debug` back to `false` before opening the server. These commands write to the
database and push events in the player's name: left open, they hand anyone who finds them a
way to fabricate transfers. The permission check still applies, but the real barrier is that
boolean, and `/phonecheck` prints a red reminder about it every time it runs.
{% endhint %}

## The three most common failures

**Photos are not saved.** Either the token in `config/upload.lua` was never filled, or
`screenshot-basic` is not started. `/phonedeps` tells you which.

**Calls are silent.** `pma-voice` is not started. The call screen still works. That is expected,
not a bug.

**The phone is two-tone in hand.** `assets/prop/` did not arrive complete during the copy: the
chassis changes colour but the camera block keeps the previous colour. See
[Streamed prop](streamed-prop.md).

## If something still does not work

Head to [Common Errors](../common-errors.md).
