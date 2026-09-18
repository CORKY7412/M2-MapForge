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
- Drag a marker to move it (clicking a marker selects its row). **Shift**+click places a new spawn even on top of an existing marker.
- The selected spawn's range shows white handles: drag an edge to set Sx or Sy, a corner to set both (the range stays centered on the spawn point). An axis only gets handles once its half-range is drawn larger than the diamond marker, so zoom in if they are missing; a point spawn (Sx = Sy = 0) has none — give it a range in the form first.
- Click a row to load it into the form (including X/Y), edit, then **Update** or press **Enter** in any field.
- X / Y / Sx / Sy / Dir / Respawn / Percent have ▲▼ stepper buttons and respond to **↑ / ↓** (hold **Shift** for ±10); steps apply live to the selected spawn. Dir stays within 0–8 and Percent within 0–100. Respawn text is left as typed until you step it; a step rewrites it as proper h/m/s with no zero parts (`120s` +1 → `2m1s`, `120s` −1 → `1m59s`, `1m` −1 → `59s`, `1h` −1 → `59m`).
- Rows the server would misread or never spawn are flagged with a **⚠** on the form fields and on the list rows: respawn values it reads differently (`120` with no unit, `1d`, `1.5h`, uppercase units, a space…, with the time it would actually use), group rows (`g` / `ga` / `r`) with Sx and Sy both 0, and vnum 0. Exports report how many remain. See [`docs/mapformat/server-regen.md`](docs/mapformat/server-regen.md).
- Delete rows (row button, or **Del** on the selected spawn), import an existing `.txt`, reset a file, or export the current file in correct 11-column server format.
- Spawns are drawn on the map as markers with their spawn-extent rectangles.

### Objects (areadata)
Per-sector `areadata.txt` editor:

- Sector dropdown; list of objects with CRC and cell position.
- Edit x/y/z, property CRC, rotation (yaw/pitch/roll) and height bias.
- Add / delete objects (row button, or **Del** on the selected object); markers plotted at (x, −y).
- Click a marker to select it — including one in another sector, which switches the sector — and drag it to move. Dragging is the only way to move an object on the map: clicking the map never relocates one (type X / Y for exact values). **Esc** deselects.
- When a move (drag, stepper or Update) takes an object into another sector, MapForge asks whether to move its record into that sector's `areadata.txt` — **Enter** confirms, **Esc** keeps it where it is.
- Double-click a row to center the map on that object with a mild zoom.
- X / Y / Z / Bias / Yaw / Pitch / Roll have ▲▼ stepper buttons and respond to **↑ / ↓**: positions step 10 cm (**Shift** 100 cm), rotations 1° (**Shift** 10°) and wrap at 360°. Steps apply live to the selected object; **Enter** in any field runs **Update**, and the button turns gold (`Update *`) while the form has unsaved edits.
- Objects the client would drop or misplace are flagged with a **⚠** on the form and the list rows — CRC 0, a position outside the sector whose `areadata.txt` holds it, fractional rotation — and exports report how many remain. See [`docs/mapformat/areadata-txt.md`](docs/mapformat/areadata-txt.md).
- Export `areadata.txt` (also bundled in the map ZIP).

### Export / import
- **Export map (.zip)** — re-zips every original file verbatim and rewrites the files you edited (regen/boss/stone/npc, areadata, generated `server_attr`, imported layers). Uses a built-in ZIP writer + CRC32, no external library.
- **Export PNG** — renders the whole current layer to a full-resolution PNG (cols×256 by rows×256).
- **Import PNG** — decodes a full-map PNG back into the current layer's per-sector files. Minimap and tile round-trip losslessly / index-exact; height, shadow and attribute are approximate; water is not supported.

### Group selection (Regen and Objects)
Hold **Ctrl** (**Cmd** on macOS) to work on several spawns or objects at once; plain left-drag still pans the map.

- **Ctrl+click** a marker or a list row adds it to the selection, or takes it out again. **Shift+click** a list row selects the range from the current row. **Ctrl+A** selects everything in the current regen file / sector.
- **Ctrl+drag** from empty map draws a box and adds what is inside; **Ctrl+Alt+drag** removes what is inside instead. Only items with a marker on the map are boxed (not `s` / `e` rows), and on the Objects tab only the current sector.
- Drag any selected marker to move the whole group by the same offset; the group stops as a block at the map edge.
- **Arrow keys** nudge the selection (one item or a group) while the focus is outside the form: spawns by 1 m, objects by 10 cm, ×10 with **Shift**. A run of presses is one undo step. Inside a form field the arrows step that field instead.
- The form shows a value where all selected items agree and stays blank where they differ. **Update** only writes the fields you filled in, so one edit (say, a respawn time or a CRC) applies to the whole group. Steppers shift every item by the same amount.
- **Del** deletes the group behind one prompt; **Esc**, a plain click on a marker, or a click on empty map drops the group — that click never places a spawn. Every group action is one undo step.
- **Ctrl+C / Ctrl+X / Ctrl+V** copy, cut and paste the selection (one item or a group). The clipboard holds plain text in the game's own format — regen rows as `regen.txt` lines, objects as `areadata.txt` blocks — so it pastes into a text editor, and lines or blocks copied from a real file paste into the map. A paste is centred on the mouse when it is over the map (otherwise nudged 2 m off the originals), kept inside the map as a block, and becomes the new selection; objects land in the sector under the paste when the map has it. Copy then paste is also how you duplicate. Inside a form field the shortcuts act on the text as usual.
- Range handles are hidden while a group is selected, and the move-to-another-sector prompt is only asked for single objects (the position warning still flags the rest).

### Tools
Optional helpers under the **Tools** menu. The ruler and the stats panel are off until you switch them on.

- **Direction arrows** (the one helper that is on by default): point spawns with Dir 1–8 get a small arrow showing where they face; Dir 0 (random) and spawns with a range get none, since the server only applies Dir to point spawns. The compass mapping follows the usual regen convention — 1 south, 2 south-east, 3 east … 8 south-west — and is not verified against the client; `DIR_BASE` / `DIR_SIGN` at the top of the script flip it.
- **Stats panel**: an overlay in the top-left of the map with rows and summed `count` for every regen file, the current file by type with its warning count and top 10 vnums, and the objects (total, sectors used, distinct and most-used CRCs, warnings). A group row counts its groups, not the mobs inside them.
- **Ruler**: click two points on the map to measure the distance in meters, with Δx / Δy (map units are meters: 1 px of the map image = 1 m). A third click starts a new measurement, **Esc** clears it. While the ruler is on, clicks only measure — nothing is placed or selected — and dragging still pans.

### Safety & UI
- **Undo / redo** for every spawn and object edit — place, move, edit, delete, import or reset a regen file, add or delete an object, move an object to another sector. **Ctrl+Z** undoes, **Ctrl+Y** or **Ctrl+Shift+Z** redoes (also as **↶ Undo / ↷ Redo** buttons in the top bar and under the **Edit** menu; both name the next step). Up to 100 steps; a run of stepper presses on one item counts as one step. Undo jumps back to the tab, file and sector where the edit happened. While a form has unsaved typed text, Ctrl+Z is left to the text field. Layer imports, `server_attr` and map merges are not tracked, and loading or closing a map clears the history.
- Confirmation modal on every destructive action (reset a regen file, delete a spawn/object, close map).
- Working menu bar: **File · Edit · Map · Server Attr · Regen · Tools**.
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
