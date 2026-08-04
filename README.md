# ROM Revver plugins

Ready-to-use plugins for [ROM Revver](https://github.com/Rom-Revver/rom-revver-releases),
plus a guide to writing your own. A plugin is just a **single `.json` file**: no coding,
nothing to compile.

## Add a plugin (30 seconds)

1. Download a `.json` file from [`plugins/`](plugins/).
2. In ROM Revver: **Menu ▸ Plugins ▸ Add plugins…** and pick the file.
3. Click its name in the **Plugins** menu to open it.

That's it. Installed plugins are remembered next time you open the app.

## What's here

Plugins come in three kinds, from safest to most powerful:

| Folder | Kind | What it does | Needs |
| --- | --- | --- | --- |
| [`plugins/display/`](plugins/display/) | **Display panel** | Shows ROM info + tables read-only. Can't change anything. | just a ROM |
| `plugins/config-mods/` *(none published yet)* | **Config mod** | Writes **values into tables, bound by ECU address**. Reversible and undoable. | a matching ROM |
| [`plugins/firmware-patches/`](plugins/firmware-patches/) | **Firmware patch** | Writes **code bytes** at fixed addresses (advanced). | a matching ROM |

Start with a **display** plugin: it can't hurt anything. Try
[`plugins/display/welcome.json`](plugins/display/welcome.json).

> ⚠️ **A note on firmware patches:** ROM Revver never tests or endorses them. What a
> patch does is entirely the plugin author's responsibility. Only install patches from
> someone you trust, and always verify before flashing. (This warning is shown, and
> can't be turned off, every time you install one.)

## What a plugin looks like

Just JSON: no build step, no code.

```json
{
  "id": "com.you.hello",
  "name": "Hello",
  "panel": {
    "blocks": [{ "type": "heading", "text": "Hello!" }]
  }
}
```

## Make your own

See **[GUIDE.md](GUIDE.md)**. It starts with the simplest possible plugin and adds
one idea at a time.

## Found a problem with a plugin here?

Open an **Issue** on this repo (the **Issues** tab). Thanks!

---

This collection is versioned independently of the app. See [`VERSION`](VERSION) and the
`v*` tags. Proprietary, © Ethan Melamed, all rights reserved. See [`LICENSE`](LICENSE).
