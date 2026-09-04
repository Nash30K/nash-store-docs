# Requirements

## Server software

- **FiveM artifacts**: latest recommended build. The resource declares `fx_version 'cerulean'`
  and `lua54 'yes'`.
- **OneSync**: required. It is a hard dependency (`/onesync` in `fxmanifest.lua`), so the
  resource does not start without it.
- **MySQL**: 5.7 or 8.0. Tables are created as `InnoDB` / `utf8mb4`.
- The MySQL account used by `oxmysql` needs `ALTER TABLE` and read access to
  `information_schema`. Both are used at every boot by the migration pass. See
  [Database](database.md).

## Required resources

| Resource | Purpose |
| -------- | ------- |
| `ox_lib` | Client/server callbacks, notification fallback |
| `oxmysql` | Database access |
| A **framework** | `es_extended` (ESX), `qb-core` (QBCore), `qbx_core` (QBOX), or your own through `server/bridge/custom.lua`. It provides the player identifier, the RP name, money and job |
| An **inventory** | Only if `Config.UseItem = true` (the default). `ox_inventory`, `qb-inventory` and derivatives, the framework's own inventory, or a custom bridge |

{% hint style="info" %}
No framework is declared as a hard dependency, and that is deliberate: a missing hard
dependency prevents a resource from starting at all. NASH Phone resolves the framework at
boot and announces it in the console instead.
{% endhint %}

## Optional resources

None of these block startup. The phone boots without them and tells you in the console what
it lost.

| Resource | What you lose without it |
| -------- | ------------------------ |
| `pma-voice` | Voice during calls. The call still rings, connects and is logged, but nobody hears anybody. The provider name is read from `Config.Voice.provider`, so a different voice resource can be named there |
| `screenshot-basic` | In-game captures: Camera photos, Snapz and ChatApp snaps. The gallery stops filling up |
| `nash-phoneprop` | The 3D model held in hand. The phone falls back to `Config.Prop.fallback` (`prop_npc_phone_02`) and stays usable. See [Streamed prop](streamed-prop.md) |
| `xsound` | Link playback in the Music app (proximity audio). The rest of the phone is untouched |

`/phonedeps` prints this exact list in game with the live state of every resource, plus a
real query against the database: a started `oxmysql` pointed at a wrong connection string
still reports as `started`.

## Outside the server

| Requirement | Purpose |
| ----------- | ------- |
| A **Fivemanage account** | Image and video hosting for the Camera. Free tier, no card. The phone stores links, not files, so hosting is not optional if you want the camera to work. See [Image hosting](image-hosting.md) |

## Framework and inventory selection

Both are chosen **separately**, in `config/bridge.lua`. An ESX server running `ox_inventory`
is the most common setup there is, and that combination needs no configuration at all.

| Key | Default | Accepted values |
|---|---|---|
| `Config.Framework` | `'auto'` | `auto`, `esx`, `qbcore`, `qbox`, `custom` |
| `Config.Inventory` | `'auto'` | `auto`, `ox`, `qb`, `framework`, `custom` |

Detection order in `auto` mode:

- **Framework**: Qbox, then QBCore, then ESX. Qbox is tried first on purpose: many Qbox
  servers run a compatibility `qb-core` alongside, and looking for QBCore first would file
  them in the wrong bridge.
- **Inventory**: `ox_inventory`, then `qb-inventory` (and `lj-inventory`), then the
  framework's own inventory. When `ox_inventory` runs, it owns the items regardless of the
  framework underneath.

Both decisions are printed at boot:

```
[nash_phone] framework détecté : qbox.
[nash_phone] inventaire détecté : ox.
```

If those two lines say what you expect, `config/bridge.lua` does not concern you.

{% hint style="danger" %}
Changing `Config.Framework` on a server that is already open orphans every phone row. All
tables are keyed on the identifier the framework returns, and that identifier has a
different shape from one framework to another (ESX license, QB citizenid). Players would
find an empty phone, and their numbers, contacts, messages and photos would become
unreachable. Decide before opening, or write a migration.
{% endhint %}

## Compatibility notes

- The phone asks an inventory only two questions: does this player carry the phone item, and
  is that item declared on the server. Nothing else.
- On **Qbox**, usable items are registered through `ox_inventory`, not through the core. On a
  Qbox server without `ox_inventory`, using the item does nothing. The key and the `/phone`
  command still open the phone for anyone who owns the item.
- A custom framework is wired by filling `server/bridge/custom.lua` and setting
  `Config.Framework = 'custom'`. A bridge that does not implement a function returns a
  neutral value and says so once in the console rather than erroring.
