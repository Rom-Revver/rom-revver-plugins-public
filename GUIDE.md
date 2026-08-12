# Making a ROM Revver plugin

A ROM Revver plugin is one `.json` file: data, never code, and nothing in it ever runs.
This guide starts at the smallest possible plugin and adds one idea per section. Read only
as far as you need: each level links back to the ones before it.

- [Level 0: the smallest plugin](#level-0-the-smallest-plugin)
- [Level 1: show some ROM data](#level-1-show-some-rom-data)
- [Pointing at a table](#pointing-at-a-table)
- [Level 2: write a value (config mod)](#level-2-write-a-value-config-mod)
- [Level 3: a firmware patch (advanced)](#level-3-a-firmware-patch-advanced)
- [Cover more than one ECU](#cover-more-than-one-ecu)
- [Organize your plugin in a folder (optional)](#organize-your-plugin-in-a-folder-optional)
- [Quick reference](#quick-reference)
- [When something does not work](#when-something-does-not-work)

---

## Level 0: the smallest plugin

Every plugin needs exactly three fields: `id`, `name`, and `panel`.

```json
{
  "id": "com.you.hello",
  "name": "Hello",
  "panel": {
    "blocks": [{ "type": "heading", "text": "Hello!" }]
  }
}
```

Save it as `hello.json`, then add it: **Menu ▸ Plugins ▸ Add plugins…**. Open it and you'll
see a panel with one heading.

- `id`: a unique, stable name for your plugin. Use reverse-domain style (`com.you.hello`) so
  it never clashes with someone else's.
- `name`: the label in the Plugins menu, and the panel's title by default.
- `panel.blocks`: the content, top to bottom. [Level 1](#level-1-show-some-rom-data) adds
  more block types.
- `panel.title`: optional. Defaults to `name` when you leave it out.

A missing `id`, `name`, or `panel`, or malformed JSON, gets the file rejected with a message.

Unknown fields and unknown block types are tolerated. ROM Revver ignores them and reports a
warning instead of a hard failure, which is what keeps an older copy of the app working with
a newer manifest. Warnings show as a note in the panel.

**Iterating on a plugin.**

- Add a plugin whose `id` matches one you already installed, and it replaces it.
- So the loop is: edit the file, add it again, see the change.
- Installed plugins are remembered across launches.
- One exception: while that plugin's mod or patch is applied to the open ROM, ROM Revver
  leaves the installed copy alone rather than swapping it. Uninstall the mod or patch first.

---

## Level 1: show some ROM data

Add more blocks to show text and live ROM data. Every block here is read-only: a display
plugin can never change your ROM.

```json
{
  "id": "com.you.info",
  "name": "My Info",
  "panel": {
    "blocks": [
      { "type": "heading", "text": "About this ROM" },
      { "type": "romInfo" },
      { "type": "text", "text": "The **idle** tables are below." },
      { "type": "tableValues", "storageaddress": "0x1234" },
      { "type": "tableLink", "storageaddress": "0x1234", "text": "Open the idle table" },
      { "type": "tableList", "category": "Idle Control" }
    ]
  }
}
```

All eight block types:

| `type` | Fields | Renders |
| --- | --- | --- |
| `heading` | `text` | A section heading. |
| `text` | `text` | A paragraph of safe Markdown (see below). |
| `divider` | none | A horizontal rule. |
| `romInfo` | none | The open ROM's calibration, vehicle, and size rows. |
| `tableList` | `category?` | The ROM's table names, clickable. Filtered to a category when given. |
| `tableValues` | `storageaddress` (required), `table?` | A table's scaled values as a read-only grid, with axis headers. |
| `tableLink` | `storageaddress` (required), `table?`, `text?` | A button that opens that table's panel. Labelled `text`, or the resolved table name when `text` is left out. |
| `pivotTable` | `xColumn`, `yColumn`, `valueColumn`, `xBreakpoints` or `xAxis`, `yBreakpoints` or `yAxis`, `aggregation?`, `interpolate?` | A read-only pivot of logged datalog samples, binned onto two axes. |

`tableValues`, `tableLink`, and a `pivotTable` axis all bind to a table the same way. That
rule gets its own section: [Pointing at a table](#pointing-at-a-table).

- **Safe Markdown in `text` blocks.** A `text` block supports a small subset: `#` through
  `######` headings, `**bold**`, `*italic*` or `_italic_`, `` `code` ``, `- ` or `* ` bullet
  lists, `1. ` numbered lists, and paragraphs separated by a blank line. Everything else
  (links, images, tables, raw HTML) renders as literal text. There's no HTML passthrough, so
  a `text` block can never inject markup or a script.
- **Data blocks need a ROM.** `romInfo`, `tableList`, `tableValues`, `tableLink`, and
  `pivotTable` all need a matching ROM open. With none, they show an "Open a ROM to see
  this" placeholder.
- **A broken `tableLink` stays visible.** If its table isn't in the open ROM, it renders
  disabled with a plain-language reason. It never silently does nothing.
- **One panel-wide check.** If any of a plugin's table references fails to resolve, the panel
  says so up front and lists each one, instead of making you discover it block by block.
- **Values are always scaled.** They come from ROM Revver's scaling engine. A plugin never
  does its own maths and never sees raw bytes.
- **Display plugins can't write.** Nothing at this level can change the ROM.
- `tableList`'s `category` is a name and must match your definition exactly.

### The pivotTable block

`pivotTable` is the one block that needs `params`, so it earns a few extra lines.

> `pivotTable` draws once a datalog is imported into the workspace — via **Menu ▸ Import
> datalog…**, or a plugin's own `read-datalog` capability (a **Load datalog** button in its
> panel; see the `capabilities` row below). Until then it shows a "no datalog" placeholder in
> place, never an empty grid.

- It plots **logged datalog samples, not ROM bytes**, and it only draws once a datalog is
  imported. Until then, it says so in place.
- `xColumn`, `yColumn`, and `valueColumn` each name the `id` of a `select` param whose
  `source` is `"datalog:columns"`. The user picks which logged signal fills each role.
- Give that param a `sensor` (e.g. `"rpm"`, or a verbose phrase like `"Coolant Temp"`) and
  its column **auto-picks** from the loaded log instead — matched the same way the coverage
  overlay matches a table axis (exact name, then a unit-suffix/alias fallback, then the
  OBD-II-standard PID names a VersaTuner-style export commonly uses — "Intake Manifold
  Absolute Pressure" resolves to `"map"` with no alias to write, since it's the official SAE
  name). The dropdown still appears when nothing plausible matches.
- **`sensor` is opt-in per param, not all-or-nothing.** A plugin built around specific
  signals can declare it on the columns with a fixed role (the worked example below
  declares `"rpm"`/`"load"` on its X/Y columns) and still leave another column free-form
  (its value column, AFR, has no `sensor` — the user always picks it) — declaring `sensor`
  on one param never obligates declaring it on the rest. A general-purpose, pick-anything
  exploration tool should leave every column's `sensor` unset — auto-picking would fight
  the whole point of letting the user choose freely.
- Each axis gets its breakpoints from one of three places:
  - a static `xBreakpoints` / `yBreakpoints` number array,
  - an `{ "storageaddress": …, "axis": "x"|"y" }` reference that aligns the pivot to a real
    table's own axis, or
  - a `{ "param": …, "axis": … }` reference naming a `rom:tables` select, so the user
    chooses their own table. See
    [Let the user pick the table instead](#let-the-user-pick-the-table-instead).

  An axis with no breakpoints is refused, not guessed at.
- A `rom:tables` select that an axis points at must declare **no `options` and no
  `default`**. The table is the user's to choose, so a manifest that pre-fills it is
  refused with a warning. What the select stores is that table's **address**, not its name.
- `aggregation` is optional: `average` (the default), `count`, `min`, `max`, `sum`, `median`,
  or `stdev`. `interpolate` is optional, and fills empty interior cells.

A worked example: three logged columns, binned onto a static grid.

```json
{
  "id": "com.example.afr-pivot",
  "name": "AFR by RPM and Load",
  "version": "1.0.0",
  "description": "A read-only pivot of logged AFR across RPM and load bins.",
  "params": [
    { "id": "rpmColumn", "label": "RPM column", "type": "select", "source": "datalog:columns", "sensor": "rpm" },
    { "id": "loadColumn", "label": "Load column", "type": "select", "source": "datalog:columns", "sensor": "load" },
    { "id": "afrColumn", "label": "AFR column", "type": "select", "source": "datalog:columns" }
  ],
  "panel": {
    "title": "AFR by RPM and Load",
    "blocks": [
      { "type": "heading", "text": "Logged AFR, binned by RPM and load" },
      {
        "type": "pivotTable",
        "xColumn": "rpmColumn",
        "yColumn": "loadColumn",
        "valueColumn": "afrColumn",
        "xBreakpoints": [800, 1600, 2400, 3200, 4000, 4800],
        "yBreakpoints": [20, 40, 60, 80, 100],
        "aggregation": "average"
      }
    ]
  }
}
```

That's everything a **display** plugin can do. See the ready examples in
[`plugins/display/`](plugins/display/). The next levels let a plugin *change* the ROM,
always behind a clear, reversible, opt-in prompt.

---

## Pointing at a table

Every block or mod write that targets a calibration table binds to it the same way. Read
this section once. Levels 1, 2, and 3 all reuse it.

| Field | Meaning |
| --- | --- |
| `storageaddress` | Required. The table's ECU storage address, exactly as its definition declares it, as a `"0x"`-prefixed hex string. |
| `table` | Optional. A tie-break only, used when one address matches several tables in the same definition. Applied after the address match. Never a binding on its own. |

- The `0x` prefix is required. Bare digits and JSON numbers are refused rather than guessed
  at, so an address can never be read in the wrong base.
- `storageaddress` alone is a complete, valid binding.
- When one address matches several tables, ROM Revver **refuses** and names every candidate.
  Add `table` to say which you meant.

**A table name is not a binding.**

- Two correct definitions for the same ECU agree on where a table lives in the binary, but
  not on what to call it.
- Resolving by name can land on the wrong table. For a plugin that writes values, that means
  writing to the wrong part of someone's car.
- A binding with no `storageaddress` is refused outright, not merely discouraged.
- Address binding is also what lets two plugins coexist, since only one definition loads at a
  time.

### Where the address comes from

1. Open the ROM **and** your definition in ROM Revver.
2. Select the table.
3. Open its ⓘ (Info) panel.
4. Click **Copy plugin binding**.

That puts `"storageaddress": "0x…"` for that exact table on your clipboard, read from the
definition you actually loaded. Paste it straight into your plugin.

- **Never hand-type an address.**
- **Never paste one copied out of another tool or another plugin.** An address authored under
  a different RAM-offset convention almost always matches nothing in your definition, and
  ROM Revver never guesses at a close match: it fails **closed** with a plain "no table at
  this address" refusal.
- "Almost always" is the honest word, not "always". A wrong address can still land on some
  other table that happens to live there, and nothing downstream can tell that apart from the
  table you meant.

### Let the user pick the table instead

If your plugin is a general **tool** rather than something aimed at one car, skip addresses.
Declare a `select` param with `source: "rom:tables"`, then point the binding at that param's
`id`.

- The dropdown lists the tables in whatever definition the user has loaded, so the choice
  can never be wrong.
- What the select stores is that table's **address**, not its name.
- Today only a pivot axis accepts `param`. Display blocks still need an address.

```jsonc
{
  "params": [
    { "id": "alignTo", "label": "Align to table", "type": "select", "source": "rom:tables" }
  ],
  "panel": {
    "blocks": [
      {
        "type": "pivotTable",
        "xColumn": "x",
        "yColumn": "y",
        "valueColumn": "v",
        "xAxis": { "param": "alignTo", "axis": "x" }, // no address, no name: the user chooses
        "yBreakpoints": [0, 1000, 2000, 3000, 4000]
      }
    ]
  }
}
```

---

## Level 2: write a value (config mod)

A **config mod** writes values into existing tables: the same thing you'd do by hand in the
editor, packaged so others can apply it in one click. It's reversible, and it lands on the
undo stack.

Two additions on top of [Level 1](#level-1-show-some-rom-data):

1. Declare the `write-tables` **capability** (permission to change tables).
2. Add a `mod` listing the cells to write.

```json
{
  "id": "com.you.idle-bump",
  "name": "Idle Bump",
  "capabilities": ["write-tables"],
  "panel": {
    "blocks": [
      { "type": "heading", "text": "Idle Bump" },
      { "type": "tableValues", "storageaddress": "0x1234" }
    ]
  },
  "mod": {
    "writes": [
      { "storageaddress": "0x1234", "cells": [{ "row": 0, "col": 0, "value": 850 }] }
    ]
  }
}
```

- `cells` are `{ row, col, value }`. `value` is the **absolute engineering value** you'd type
  in the editor (850 RPM here), never a delta. Out-of-range values are clamped. To move a
  cell relative to what it currently holds, use a parameterized mod with the `offset` op (see
  below).
- Opening the panel shows an **Install mod** button. Installing shows a prompt naming exactly
  what will change. **Uninstall mod** puts the original values back, byte for byte. Both are
  undoable and go through the same checksum-verified write path as a manual edit.
- A `mod` (or a `patch`) parses fine without its capability declared. Install then just stays
  disabled, with a reason naming the missing capability. The gate lives at install time in
  the app, not in the file format.
- `storageaddress` is what binds the write to a table. A table name alone is never a binding:
  see [Pointing at a table](#pointing-at-a-table) for the full rule.
- `storageaddress` also decides which cars the mod works on, and the answer is not "all of
  them": read [Cover more than one ECU](#cover-more-than-one-ecu) before you share it.

### Let the user pick a value (parameterized mods)

Instead of a hard-coded `value`, a mod can expose a **numeric input** and compute what to
write from it.

- Declare a `params` entry, then use a *computed* write:
  `{ storageaddress, table?, op, param, range and/or cells }`.
- `params` shape: `[{ id, label, min, max, default, unit?, step? }]`.
- `storageaddress` is required on a computed write too.
- A computed write **must declare which cells it targets**: a `range`, a `cells` list, or
  both. It never touches the whole table implicitly. Naming the cells is how a mod states its
  own blast radius and stays auditable. A computed write with neither, or one whose cells
  fall outside the table, is **refused outright**.

```jsonc
{
  "capabilities": ["write-tables"],
  "params": [
    { "id": "idle", "label": "Target idle", "unit": "RPM", "min": 800, "max": 1300, "default": 950, "step": 50 }
  ],
  "mod": {
    "writes": [
      {
        "storageaddress": "0x1234",
        "table": "Idle Target - Base",
        "op": "raiseTo",
        "param": "idle",
        "range": { "from": { "row": 4, "col": 0 }, "to": { "row": 6, "col": 0 } }
      }
    ]
  }
}
```

`table` here is doing something specific and optional: it's a **tie-break**, applied only if
`storageaddress` matches more than one table in your definition (some defs alias storage
between tables). It's never required and never the primary key: `storageaddress` alone is a
complete, valid binding.

The ops, a fixed vocabulary (nothing here is executed as code), each applying only to the
**declared** cells:

| `op` | Does |
| --- | --- |
| `raiseTo` | A **floor**. Raises a declared cell below the value up to it. Never lowers. |
| `lowerTo` | A **cap**. Lowers a declared cell above the value down to it. Never raises. |
| `set` | Sets each declared cell to the value. |
| `offset` | Adds the value to each declared cell. |

Declaring the targets:

- `range`: `{ "from": {row,col}, "to": {row,col} }`, inclusive of both corners, in either
  order. `from row 4 to row 6` means rows 4, 5, **and** 6. The install prompt reports how
  many values change, not which rows, so check the range against what you meant.
- `cells`: an explicit `[{ row, col }, …]` list, with **no** `value` (the value is computed
  from the op and the input). Best for scattered cells.

The panel shows a number input bounded by `min`/`max`. Install writes only the declared cells
that actually change, so a lower `raiseTo` value touches fewer of them. Install and Uninstall
stay reversible.

There's no published config-mod example to copy, and that's deliberate: a config mod targets
real tables, so any worked example would carry one car's addresses. The complete shape is the
Level 2 example above. The only part you supply is the `storageaddress`, and
[Where the address comes from](#where-the-address-comes-from) gets you one from your own
definition in seconds.

---

## Level 3: a firmware patch (advanced)

A **firmware patch** writes raw **code bytes** at fixed ROM addresses, for a feature that
doesn't exist as a table. This is the deep end.

> 🛑 **ROM Revver never tests, verifies, or endorses firmware patches.** A patch can change
> how the ECU runs in ways the app can't check. What it does is entirely your responsibility
> as the author. Only share patches you have validated yourself. This warning is shown,
> unremovable, every time anyone installs one.

Two additions on top of [Level 2](#level-2-write-a-value-config-mod):

1. Declare the `patch-firmware` capability.
2. Add a `patch` with one or more **regions**. Each region has an `address`, the `bytes` to
   write, and the `original` bytes expected there first, as equal-length hex strings.

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

An address is a `"0x"`-prefixed hex string, the same rule as `storageaddress`: a bare digit
string or a JSON number is refused rather than guessed at.

- **`original` is a safety anchor.** ROM Revver refuses to install unless the ROM currently
  holds exactly those bytes, so a patch can never be applied to the wrong ROM. Whether it's
  installed is read back from the bytes, never session bookkeeping, so it can never get out
  of sync. Never widen or omit `original` to make an install go through: the refusal is the
  feature.
- **Find the right address yourself.** A code blob needs genuine **free space** (usually
  erased flash, all `FF`), so `original` is then all `ff…`. Finding safe free space, and
  hooking existing code to run your bytes for a working feature, is real firmware work this
  guide doesn't cover, and it can only be proven in a car.
- **Install** shows the loudest prompt: the untested warning, the exact byte-by-byte preview,
  and, for a per-ECU patch, which ECU it matched. **Uninstall** restores the originals
  exactly.

Working (inert, safe) example:
[`plugins/firmware-patches/freespace-marker.json`](plugins/firmware-patches/freespace-marker.json).

---

## Cover more than one ECU

This is the honest-limitations section. Every number and every caveat here matters.

- **Config mods bind by storage address.** An address is neither universal nor unique to one
  calibration: it's shared by however many calibrations put that table in the same place.
- In the real RX-8 definition, `Idle Target - Base` sits at **six** different addresses, and
  the ~30 calibration ids the definition covers cluster onto those six in groups of **9, 8,
  5, 4, 2, and 2**.
- So a mod you author against one calibration also works on every calibration in its group,
  **and only on those**. There's no way to tell which from the outside. You check each
  definition.

The two outcomes on a calibration you didn't author for:

- **The address matches no table.** The plugin fails closed. The panel says the binding
  didn't resolve, and Install is refused. Nothing can be written.
- **The address holds a different table.** The mod resolves to that table and offers to
  install. The optional `table` name does **not** protect you here.

The two guards, and you should rely on both:

- ROM Revver's install prompt **names the table it resolved to, under the user's own
  definition**. They should read it before confirming.
- As the author, name and describe your plugin with the calibrations you actually verified.
  A `mod` has no way to declare its ECU: there's no `perEcu` form for `mod` today, only for
  `patch`.

Firmware patches use raw addresses too, and free space sits at a different address in each
calibration. Unlike `mod`, `patch` **does** have a `perEcu` map keyed by calibration id (the
definition's `xmlid`, or the matched calibration string), so one plugin can cover several:

```json
{
  "capabilities": ["patch-firmware"],
  "patch": {
    "perEcu": {
      "60E1B900": [
        { "address": "0x6A1DC", "bytes": "524f4d5245565652", "original": "ffffffffffffffff" }
      ],
      "60E1A500": [
        { "address": "0x6A2B0", "bytes": "524f4d5245565652", "original": "ffffffffffffffff" }
      ]
    }
  }
}
```

The bytes in that example are the same inert `ROMREVVR` marker as
[`plugins/firmware-patches/freespace-marker.json`](plugins/firmware-patches/freespace-marker.json),
written into erased flash. **The two addresses are illustrative only.**

Free space sits somewhere different in each calibration, and finding it is your job. Do not
copy an address out of this guide, or out of another calibration's entry, and expect it to be
free in yours.

On install, ROM Revver picks the entry matching the open ROM and **refuses if your ECU isn't
listed**. It will never write one ECU's bytes to another.

You still work out each ECU's addresses yourself (per-ECU precompiling). ROM Revver just
picks the right set.

---

## Organize your plugin in a folder (optional)

If you keep your plugin in a subfolder of your own repo (say, one folder per car), declare
that path so the marketplace browse list mirrors it:

```jsonc
{
  "id": "com.you.idle-bump",
  "name": "Idle Bump",
  "browsePath": "mazda/rx8",
  "panel": { "blocks": [{ "type": "heading", "text": "Idle Bump" }] }
}
```

`browsePath` is an author-chosen, `/`-joined path, nested arbitrarily deep. Plugins with no
`browsePath` show at the top level of the browse list.

---

## Quick reference

| Field | Required? | Meaning |
| --- | --- | --- |
| `id` | Required | Unique, stable id (reverse-domain style). |
| `name` | Required | Menu label and default panel title. |
| `panel.blocks` | Required | Panel content, top to bottom. See [Level 1](#level-1-show-some-rom-data). |
| `panel.title` | Optional | The panel's title. Defaults to `name`. |
| `version` | Optional | Informational inside the app. Required to list the plugin in the marketplace. |
| `description` | Optional | A one-line summary shown in the marketplace browse list. Required to list the plugin there, same as `version`. |
| `minAppVersion` | Optional | `MAJOR.MINOR.PATCH`. The oldest app version that may run this plugin. The marketplace shows a listing below that floor as incompatible instead of offering Install. A malformed value is dropped with a warning, never a hard failure. |
| `targetAppVersion` | Optional | `MAJOR.MINOR.PATCH` you authored against. Informational only. Malformed values are handled the same way as `minAppVersion`. |
| `capabilities` | Required for writes, and for the datalog-load button | `write-tables` for a config mod, `patch-firmware` for a firmware patch, `read-datalog` to offer the app-owned "Load datalog" button in your panel (a read capability — nothing is written). `read-rom` is also accepted, but nothing gates on it today: every display block already works without declaring any capability. |
| `mod` | Required for config mods | Fixed: `{ writes: [{ storageaddress, cells: [{row,col,value}] }] }`. Computed: `{ writes: [{ storageaddress, op, param, range\|cells }] }` (`table` optional, tie-break only). See [Level 2](#level-2-write-a-value-config-mod). |
| `params` | Required for parameterized mods | `[{ id, label, min, max, default, unit?, step? }]`. A numeric input a computed write reads. |
| `patch` | Required for firmware patches | `{ regions: […] }` or `{ perEcu: {…} }`. See [Level 3](#level-3-a-firmware-patch-advanced) and [Cover more than one ECU](#cover-more-than-one-ecu). |
| `browsePath` | Optional | A `/`-joined folder path (for example `"mazda/rx8"`) matching where you keep this plugin in its source tree. The marketplace browse tree groups plugins by it, nested arbitrarily deep. Leave it out to show the plugin at the top level. |

Unknown fields and block types are ignored with a warning, so a newer plugin never hard-fails
on an older app. When something doesn't show up, open the panel and read the **warnings**
note.

---

## When something does not work

- **`"… is not a valid plugin"`**: the JSON is malformed, or `id`, `name`, or `panel` is
  missing.
- **`"Added plugin … (N warnings)"`**: it loaded, but N things were ignored. Open the panel
  and expand the warnings note.
- **A data block says "Open a ROM to see this"**: open a matching ROM first.
- **A binding won't resolve**: the panel lists every reference it couldn't match. Check the
  address carries the `0x` prefix and is your definition's own `storageaddress`. If it
  matches several tables, add `table`.
- **Install is disabled on a mod or patch**: the matching capability isn't declared.
- **Re-adding your edited file says "left alone (mod/patch applied)"**: ROM Revver won't swap
  a plugin's file out from under a mod or patch that is currently applied to the open ROM.
  Uninstall it first, then add the edited file.
- **A `pivotTable` only ever shows a placeholder**: no datalog is loaded yet. Import one —
  **Menu ▸ Import datalog…**, or your panel's own **Load datalog** button if you declared
  `read-datalog` — and the pivot draws from it.
