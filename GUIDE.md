# Making a ROM Revver plugin

A plugin is one `.json` file. This guide starts with the smallest possible plugin and
adds **one idea per section**. Read only as far as you need — each level links back to
the ones before it.

- [Level 0 — the smallest plugin](#level-0--the-smallest-plugin)
- [Level 1 — show some ROM data](#level-1--show-some-rom-data)
- [Level 2 — write a value (config mod)](#level-2--write-a-value-config-mod)
- [Level 3 — a firmware patch](#level-3--a-firmware-patch-advanced)
- [Cover more than one ECU](#cover-more-than-one-ecu)
- [Quick reference](#quick-reference)

---

## Level 0 — the smallest plugin

Every plugin needs exactly three things: an `id`, a `name`, and a `panel`.

```json
{
  "id": "com.you.hello",
  "name": "Hello",
  "panel": {
    "blocks": [{ "type": "heading", "text": "Hello!" }]
  }
}
```

Save it as `hello.json`, add it (**Menu ▸ Plugins ▸ Add plugins…**), and open it. You'll
see a panel with one heading.

- `id` — a unique name for your plugin. Use reverse-domain style (`com.you.hello`) so it
  never clashes with someone else's.
- `panel.blocks` — the content, top to bottom. The next level adds more block types.

If the file has a typo, ROM Revver still adds the plugin and shows a **warnings** note in
the panel telling you what it ignored — so you can fix it and re-add.

---

## Level 1 — show some ROM data

Add more **blocks** to show text and live ROM data. These are all read-only — a display
plugin can never change your ROM.

```json
{
  "id": "com.you.info",
  "name": "My Info",
  "panel": {
    "blocks": [
      { "type": "heading", "text": "About this ROM" },
      { "type": "romInfo" },
      { "type": "text", "text": "The **fuel** tables are below." },
      { "type": "tableValues", "table": "Idle Target - Base" },
      { "type": "tableList", "category": "Idle Control" }
    ]
  }
}
```

The blocks you can use:

| `type` | Shows |
| --- | --- |
| `heading` | A section title (`text`). |
| `text` | A paragraph — supports simple **bold**, *italic*, `code`, and `- ` lists. |
| `divider` | A horizontal line. |
| `romInfo` | The open ROM's calibration / vehicle / size. |
| `tableList` | Table names (all, or one `category`). |
| `tableValues` | A named table's values as a read-only grid (`table`). |

`tableValues` and `tableList` need a **matching ROM open**; with none, they show a gentle
"Open a ROM to see this" placeholder. Table and category names must match your definition
exactly. See the ready examples in [`plugins/display/`](plugins/display/).

That's everything a **display** plugin can do. The next levels let a plugin *change* the
ROM — always behind a clear, reversible, opt-in prompt.

---

## Level 2 — write a value (config mod)

A **config mod** writes values into existing tables — the same thing you'd do by hand in
the editor, packaged so others can apply it in one click. It's reversible and lands on the
undo stack.

Two new pieces on top of a [Level 1](#level-1--show-some-rom-data) plugin:

1. Declare the `write-tables` **capability** (permission to change tables).
2. Add a `mod` listing the table cells to write.

```json
{
  "id": "com.you.idle-bump",
  "name": "Idle Bump",
  "capabilities": ["write-tables"],
  "panel": {
    "blocks": [
      { "type": "heading", "text": "Idle Bump" },
      { "type": "tableValues", "table": "Idle Target - Base" }
    ]
  },
  "mod": {
    "writes": [
      { "table": "Idle Target - Base", "cells": [{ "row": 0, "col": 0, "value": 850 }] }
    ]
  }
}
```

- `cells` are `{ row, col, value }` — `value` is the **engineering value** you'd type in the
  editor (850 RPM here), not raw bytes. Out-of-range values are clamped.
- Opening the panel shows an **Install mod** button. Installing shows a prompt naming exactly
  what will change; **Uninstall mod** puts the original values back, byte-for-byte.
- Because it names the table by text, **one config mod works on every calibration** whose
  definition has that table — no per-ECU work needed. (See
  [Cover more than one ECU](#cover-more-than-one-ecu).)

### Let the user pick a value (parameterized mods)

Instead of hard-coded `value`s, a mod can expose a **numeric input** and compute what to write
from it. Declare a `params` entry, then use a *computed* write — `{ table, op, param, … }` —
that applies an operation to the input. A computed write **must declare which cells it targets**
(a `range`, a `cells` list, or both): a parameterized mod names its cells so it stays auditable
and can only ever touch what it declares — never a whole table by surprise.

```jsonc
{
  "capabilities": ["write-tables"],
  "params": [
    { "id": "idle", "label": "Target idle", "unit": "RPM", "min": 800, "max": 1300, "default": 950, "step": 50 }
  ],
  "mod": {
    "writes": [
      {
        "table": "Idle Target - Base",
        "op": "raiseTo",
        "param": "idle",
        "range": { "from": { "row": 4, "col": 0 }, "to": { "row": 6, "col": 0 } }
      }
    ]
  }
}
```

The ops (a fixed set — nothing is executed as code); each applies only to the **declared**
cells:

| `op` | Does |
| --- | --- |
| `raiseTo` | a **floor** — raises a declared cell below the value up to it; never lowers. |
| `lowerTo` | a **cap** — lowers a declared cell above the value down to it; never raises. |
| `set` | sets each declared cell to the value. |
| `offset` | adds the value to each declared cell. |

Declaring the targets:

- **`range`** — `{ "from": {row,col}, "to": {row,col} }`, inclusive of both corners (either
  order). Best for a contiguous span (like the warm end of a 1-D idle curve).
- **`cells`** — an explicit `[{ row, col }, …]` list. Best for scattered cells. (Note: no
  `value` here — the value is computed from the op + input.)
- A computed write with neither, or one whose cells fall outside the table, is refused.

The panel shows a number input (bounded by `min`/`max`); the user picks a value, and Install
writes only the declared cells that actually change (so a lower `raiseTo` value touches fewer
of them). Install/Uninstall are reversible exactly as before.

Working example — an **Adjustable Idle** mod (pick 800–1300 RPM; at 1300 it reproduces a real
tune verified byte-for-byte):
[`plugins/config-mods/adjustable-idle.json`](plugins/config-mods/adjustable-idle.json).

---

## Level 3 — a firmware patch (advanced)

A **firmware patch** writes raw **code bytes** at fixed ROM addresses — for a feature that
doesn't exist as a table. This is the deep end.

> 🛑 **ROM Revver never tests, verifies, or endorses firmware patches.** A patch can change
> how the ECU runs in ways the app can't check — what it does is entirely your
> responsibility as the author. Only share patches you have validated yourself. This
> warning is shown, unremovable, every time anyone installs one.

Two new pieces on top of [Level 2](#level-2--write-a-value-config-mod):

1. Declare the `patch-firmware` capability.
2. Add a `patch` with one or more **regions**. Each region has an `address`, the `bytes` to
   write, and the `original` bytes expected there first — as equal-length hex strings.

```json
{
  "id": "com.you.marker",
  "name": "Free-space marker",
  "capabilities": ["patch-firmware"],
  "panel": { "blocks": [{ "type": "heading", "text": "Free-space marker" }] },
  "patch": {
    "regions": [
      { "address": "0x6A1DC", "bytes": "524f4d5245565652", "original": "ffffffffffffffff" }
    ]
  }
}
```

- **`original` is a safety anchor.** ROM Revver refuses to install unless the ROM currently
  holds exactly those bytes — so a patch can never be applied to the wrong ROM. Whether it's
  installed is read back from the bytes, so it never gets out of sync.
- **Find the right address yourself.** For a code blob you need genuine **free space**
  (usually erased flash, all `FF`); the `original` is then all `ff…`. Finding safe free space
  and, for a working feature, hooking existing code to run your bytes, is real firmware work
  that this guide doesn't cover — and can only be proven in a car.
- **Install** shows the loudest prompt: the untested warning, the exact byte-by-byte preview,
  and (for per-ECU patches) which ECU it matched. **Uninstall** restores the originals exactly.

Working (inert, safe) example:
[`plugins/firmware-patches/freespace-marker.json`](plugins/firmware-patches/freespace-marker.json).

---

## Cover more than one ECU

The RX-8 definition covers many ECU calibrations (60E1B900, N3ZBEH, …). How a plugin spans
them depends on its kind:

- **Config mods** ([Level 2](#level-2--write-a-value-config-mod)) already span every ECU — they
  name tables, and each calibration's definition resolves that name. Nothing extra to do.
- **Firmware patches** ([Level 3](#level-3--a-firmware-patch-advanced)) use raw addresses, and
  free space sits at a different address in each calibration. So instead of one `regions` list,
  use a **`perEcu` map** keyed by calibration id:

```json
{
  "capabilities": ["patch-firmware"],
  "patch": {
    "perEcu": {
      "60E1B900": [{ "address": "0x6A1DC", "bytes": "…", "original": "ffff…" }],
      "N3ZBEH":   [{ "address": "0x6A1DC", "bytes": "…", "original": "ffff…" }]
    }
  }
}
```

On install, ROM Revver picks the entry matching the open ROM and **refuses if your ECU isn't
listed** — it will never write one ECU's bytes to another. You still work out each ECU's
addresses yourself (per-ECU precompiling); ROM Revver just picks the right set.

---

## Quick reference

| Field | Required? | Meaning |
| --- | --- | --- |
| `id` | ✅ | Unique, stable id (reverse-domain). |
| `name` | ✅ | Menu label + default panel title. |
| `panel.blocks` | ✅ | Panel content (see [Level 1](#level-1--show-some-rom-data)). |
| `version` | — | Optional, informational. |
| `capabilities` | for writes | `write-tables` (config mod) and/or `patch-firmware`. |
| `mod` | for config mods | fixed `{ writes: [{ table, cells: [{row,col,value}] }] }`, or computed `{ writes: [{ table, op, param, range\|cells }] }` — [Level 2](#level-2--write-a-value-config-mod). |
| `params` | for parameterized mods | `[{ id, label, min, max, default, unit?, step? }]` — a numeric input a computed write reads. |
| `patch` | for firmware patches | `{ regions: […] }` or `{ perEcu: {…} }` — [Level 3](#level-3--a-firmware-patch-advanced) / [multi-ECU](#cover-more-than-one-ecu). |

Unknown fields and block types are ignored with a warning, so a newer plugin never hard-fails
on an older app. When something doesn't show up, open the panel and read the **warnings** note.
