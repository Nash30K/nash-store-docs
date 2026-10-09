# Streamed Prop

The phone held in the player's hand is a separate resource, `nash-phoneprop`. It is a soft
dependency: without it the phone still works, but everyone holds a stock GTA handset.

## Install

1. Drop `nash-phoneprop` in `resources/`.
2. Start it **before** `nash_phone`:

```cfg
ensure nash-phoneprop
ensure nash_phone
```

{% hint style="warning" %}
Order matters. The model is registered as an archetype through `DLC_ITYP_REQUEST`. When the
phone is taken out, it waits up to 2.5 seconds for the model to load; if the model is not known,
the phone falls back to `Config.Prop.fallback` and prints one line in the player's F8 console
(see [Troubleshooting](#troubleshooting)).
{% endhint %}

## What the prop resource contains

```
nash-phoneprop/
├── fxmanifest.lua
└── stream/
    ├── custom_phone_prop.ydr    # the model
    ├── custom_phone_prop.ytd    # its textures, including the shell
    └── custom_phone_prop.ytyp   # the archetype definition
```

Its manifest declares the archetype explicitly:

```lua
this_is_a_map 'yes'
data_file 'DLC_ITYP_REQUEST' 'stream/*.ytyp'

files {
    'stream/*.ydr',
    'stream/*.ytd',
    'stream/*.ytyp',
}
```

The `stream/` folder is scanned by the streamer on its own, but the `.ytyp` has a second
consumer (the `data_file` line above), so the `files` block is not decorative. The `.ytd`
belongs there too: it carries the shell textures.

## Configuration

Everything is under `Config.Prop` in `config/main.lua`.

| Key | Default | Description |
|---|---|---|
| `model` | `'custom_phone_prop'` | Archetype streamed by `nash-phoneprop` |
| `fallback` | `'prop_npc_phone_02'` | Stock GTA model used when the archetype is unavailable |
| `bone` | `28422` | Right hand (`IK_R_Hand`) |
| `offset` | `vec3(0.0, 0.0, 0.0)` | Attachment offset |
| `rotation` | `vec3(0.0, 0.0, 0.0)` | Attachment rotation |
| `skin.enabled` | `true` | Coloured shell following the player's setting |
| `skin.txd` | `'custom_phone_prop'` | Texture dictionary inside the `.ytd` |
| `skin.textures` | `body = 'Warm_Terracotta'`, `light = 'Warm_Terracotta_light'`, `back = 'back_iphone'` | The three textures of the model that carry the colour. Only change them if the model is rebuilt |
| `skin.colors` | 4 entries | One entry per colour offered in Settings |
| `skin.default` | `'cosmic'` | Colour applied when the player has not chosen |

The phone deliberately does **not** call `IsModelValid` before requesting the model: for an
archetype registered through `DLC_ITYP_REQUEST`, that native commonly answers "no" while
`RequestModel` loads the model without trouble. Using it as a filter would send every server
to the fallback prop.

## The coloured shell

The chassis follows Settings → Display → Frame colour, live, with no restart and without
recreating the prop.

There is only one model. The three textures that carry the colour are swapped at runtime in
the prop's dictionary: the two flat tones are generated in memory, and the camera block comes
from a pre-recoloured PNG because it has a pattern. Those PNGs live in `nash_phone`, not in
the prop resource:

```
nash_phone/assets/prop/
├── back_deepblue.png
├── back_cosmic.png
├── back_silver.png
└── back_black.png
```

They are declared in the manifest as `'assets/prop/*.png'`. Without that declaration the file
is never served to the client and the back of the phone stays on the original colour.

{% hint style="info" %}
The swap applies to the pair (dictionary, texture), with no notion of entity, so it is
**local to the viewer**. On a player's screen every phone takes their colour, including other
players'. This is a deliberate trade-off: it avoids streaming one model per colour.
{% endhint %}

### Changing a colour

The four colours are fixed: `deepblue`, `cosmic`, `silver` and `black`. The swatches shown
in Settings are part of the compiled interface, and the server only accepts those four values,
so you cannot add a colour or rename one. Keep the four identifiers of
`Config.Prop.skin.colors` as they are.

What you can change is the **shade** of each one: edit `body`, `light` and the `back` PNG of
an existing entry. Nothing to rebuild, a restart of the resource is enough.

If one of the four entries is missing from `Config.Prop.skin.colors`, the swatch still exists in
Settings and the on-screen chassis changes, but the prop in hand keeps the previous shell.

### The pairing rule

`light` must always be **lighter** than `body`. It is the flat tone of the faces that catch
the light; if it drops below its `body`, the area meant to brighten darkens instead and the
volume of the chassis reads inverted. Nothing breaks and nothing warns. The prop just looks
dull.

## Troubleshooting

**Everyone holds a stock GTA phone.**

The player's F8 console prints this once per session:

```
[nash_phone] modèle « custom_phone_prop » introuvable, repli sur « prop_npc_phone_02 ».
La ressource du prop est-elle démarrée AVANT nash_phone ?
```

Check that `nash-phoneprop` is in `resources/`, started, and started before `nash_phone`.

For a closer look, any player can type `/nashprop`. It prints in the F8 console the state of `nash-phoneprop`, whether the model loads and how long it took, the
fallback model, the colour applied, and whether each `back` PNG of `Config.Prop.skin.colors`
is found.

**The phone is two-tone in hand.**

`assets/prop/` did not arrive complete during the copy. The chassis changes colour but the
camera block keeps the previous colour's PNG, on that colour only. Copy the folder again.

**A leftover file.**

`assets/prop/back_gold.png` is the remains of a colour that was removed from the list.
Nothing references it any more, but the `assets/prop/*.png` pattern still makes every client
download it. You can delete it to lighten the resource.
