# MapForge — Metin2 Map Editor (Web)

A browser-based reimplementation and modernization of the classic C# "Map Converter" (WinForms, ~2012) for Metin2 2D maps. No install, no server — everything runs client-side. Open a map folder, inspect every layer, edit spawns and objects, and export the map back out.

Runs best in **Chrome / Edge** (Chromium), which support the File System Access API for reading a map folder directly.

---

## Features

### Layer viewer
Renders all map layers straight from their binary/text sources:

- **Minimap** — decoded `minimap.dds` tiles (full DDS decoder: uncompressed 32/16-bit + DXT1/3/5). Default layer.
- **Height** — grayscale terrain from `height.raw` (131×131 uint16 vertex grid per sector).
- **Tile** — ground-texture index map from `tile.raw`.
- **Shadow** — baked `shadowmap.raw` / `shadowmap.dds`.
- **Attribute** — client `attr.atr` collision/gameplay grid.
- **Water** — water cells from `water.wtr`.

Viewport supports wheel-zoom, drag-pan, **Fit**, live attribute-cell coordinates, and a sector grid overlay.

### Attribute flags
All 8 `attr.atr` bit flags are color-coded. Base layer shows the dominant flag per cell; **Highlight attribute flags** overlays every flag on top of any layer:

| Bit | Flag | Color |
|-----|----------|--------|
| 0x01 | Block | red |
| 0x02 | Water | blue |
| 0x04 | Safezone | green |
| 0x08 | Banshop | yellow |
| 0x10 | Flag 5 | orange |
| 0x20 | Flag 6 | purple |
| 0x40 | Flag 7 | cyan |
| 0x80 | Object | magenta |

### Server attributes (`server_attr`)
- **Import from server_attr** — reads the binary, LZO1X-decompresses each block, and reconstructs the per-sector `attr.atr` grids onto the Attribute layer.
- **Generate server_attr** — builds `server_attr` from the map's `attr.atr` files (full byte preserved, 2×2 upsample, y-major, LZO1X-compressed). Downloads it and bundles it in the map ZIP.

### Regen Creator
Load, place, edit and export server spawn files — `regen.txt`, `boss.txt`, `stone.txt`, `npc.txt`:

- Click the map to place a spawn using the current form values (type, vnum, sx/sy, dir, respawn, percent, count).
- Click a row to load it into the form, edit, then **Update**.
- Delete rows, import an existing `.txt`, reset a file, or export the current file in correct 11-column server format.
- Spawns are drawn on the map as markers with their spawn-extent rectangles.

### Objects (areadata)
Per-sector `areadata.txt` editor:

- Sector dropdown; list of objects with CRC and cell position.
- Edit x/y/z, property CRC, rotation (yaw/pitch/roll) and height bias.
- Add / delete objects, select-then-click-to-move on the map, markers plotted at (x, −y).
- Export `areadata.txt` (also bundled in the map ZIP).

### Export / import
- **Export map (.zip)** — re-zips every original file verbatim and rewrites the files you edited (regen/boss/stone/npc, areadata, generated `server_attr`, imported layers). Uses a built-in ZIP writer + CRC32, no external library.
- **Export PNG** — renders the whole current layer to a full-resolution PNG (cols×256 by rows×256).
- **Import PNG** — decodes a full-map PNG back into the current layer's per-sector files. Minimap and tile round-trip losslessly / index-exact; height, shadow and attribute are approximate; water is not supported.

### Safety & UI
- Confirmation modal on every destructive action (reset a regen file, delete a spawn/object, close map).
- Working menu bar: **File · Map · Server Attr · Regen · Utility**.
- Resizable inspector sidebar.
- Loads a folder via the picker **or** by drag-and-drop onto the viewport.

---

## Supported file formats

Parsed and/or written (modern "40k-era" map format):

`setting.txt` · `areadata.txt` · `height.raw` · `tile.raw` · `attr.atr` · `water.wtr` · `shadowmap.raw` / `.dds` · `minimap.dds` · `regen.txt` · `boss.txt` · `stone.txt` · `npc.txt` · `server_attr`

---

## Browser support

| Browser | Open folder | Drag-drop | Export ZIP / files |
|---------|:-----------:|:---------:|:------------------:|
| Chrome / Edge | ✅ picker + drag | ✅ | ✅ |
| Firefox / Safari | ⚠️ drag-drop only | ✅ | ✅ (downloads) |

The File System Access folder **picker** is Chromium-only. Other browsers can still load a map by dragging its folder in, and save via downloads.

---

## Notes & limitations

- `server_attr` **Generate** uses store-mode LZO1X (all-literal, valid stream): larger than the game's own output but byte-correct on decompress — the server reads it fine.
- Non-minimap/non-tile PNG re-imports are lossy by nature (a PNG is just pixels).
- ZIP entry paths are lowercased; Metin2 map files are conventionally lowercase, so this is safe in practice.
- Very large maps (2100×2100 range) are memory/CPU heavy in a single tab.
