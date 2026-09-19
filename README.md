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

### Paint attributes
**Converter › Paint attributes…** opens a floating palette over the map (drag it by its title bar; **Esc** or × closes it). While it is open the flag overlay is forced on over whichever layer you are looking at — switch between minimap, height, tile or water to trace against — left-drag paints, and right / middle drag or holding **Space** pans.

- All 8 flags are there; tick one or several. **Set** turns them on and leaves the cell’s other flags alone, **Clear** turns them off, **Replace** makes the cell exactly the ticked flags. The Water flag only marks cells — visible water comes from `water.wtr`.
- Tools: **Brush** (round or square tip, 1–64 m, no gaps on fast drags), **Rectangle** and **Circle** (drag to size, filled, with a preview) and **Pick** (take a cell’s flags). One cell is 1 m, the same grid as spawn coordinates; strokes cross sector borders freely and cells outside the map are ignored.
- **Batch paint, by texture name** — paints the whole map in one go with the mapper’s trick: every tile whose ground texture file name starts with `stone` gets Block, every one starting with `tile` gets Safezone (prefixes are editable, comma-separated, case ignored, folders ignored). Block and Safezone each have their own **Add** and **Rebuild**, so one can be run without touching the other: **Add** only sets that flag; **Rebuild** also clears that flag wherever the texture does not match, and reports how many cells it cleared. It needs the map’s textureset, and the **Add** / **Rebuild** buttons stay disabled until one is loaded: the section shows the file name in use (`✓ metin2_c1.txt · 12 textures`) or, while it is missing, the name `setting.txt` asks for. It is picked up by itself when it sits in the map folder; otherwise press **load…** or drop the `.txt` (or the whole textureset folder) onto the palette — or anywhere on the page while the palette is open. A textureset loaded with the picker or by drag and drop is remembered by the workspace (Chrome / Edge) and loaded again when you reopen it; it belongs to that one map, so it is not carried over to other maps.
- Each stroke (and each texture run) is one undo step. Edits are written into the map’s `attr.atr` files, so **Export map** and **Generate server_attr** use them; the palette reminds you when `server_attr` is out of date and has the button to regenerate it.
- Keys: **B** brush · **R** rectangle · **C** circle · **I** pick · **X** swap Set / Clear · **[ ]** size (Shift ×8) · **1–8** toggle flags.
- The flag overlay now blends the colours of every flag a cell has (it used to show only the first), so mixed cells are visible.

### Server attributes (`server_attr`)
- **Import from server_attr** — reads the binary, LZO1X-decompresses each block, and reconstructs the per-sector `attr.atr` grids onto the Attribute layer.
- **Generate server_attr** — builds `server_attr` from the map's `attr.atr` files (full byte preserved, 2×2 upsample, y-major, LZO1X-compressed). Downloads it and bundles it in the map ZIP.

### Regen Creator
Load, place, edit and export server spawn files — `regen.txt`, `boss.txt`, `stone.txt`, `npc.txt`:

- **Load regen folder…** (button on the Regen tab, also under **File** and **Regen**) picks a folder and loads `regen.txt`, `boss.txt`, `stone.txt` and `npc.txt` from it in one go. Client maps often ship without spawn files, so point it at the server-side map folder. Only the top level of the folder is read; a file that is not there leaves the current one untouched, and the whole load is one undo step. You can also **drag and drop** the spawn files (any of the four, loose) or the server-side folder anywhere onto the page; a dropped `mob_names.txt` loads the names.
- Click the map to place a spawn using the current form values (type, vnum, sx/sy, dir, respawn, percent, count).
- Drag a marker to move it (clicking a marker selects its row). **Shift**+click places a new spawn even on top of an existing marker.
- The selected spawn's range shows white handles: drag an edge to set Sx or Sy, a corner to set both (the range stays centered on the spawn point). An axis only gets handles once its half-range is drawn larger than the diamond marker, so zoom in if they are missing; a point spawn (Sx = Sy = 0) has none — give it a range in the form first.
- Click a row to load it into the form (including X/Y), edit, then **Update** or press **Enter** in any field.
- X / Y / Sx / Sy / Dir / Respawn / Percent have ▲▼ stepper buttons and respond to **↑ / ↓** (hold **Shift** for ±10); steps apply live to the selected spawn. Dir stays within 0–8 and Percent within 0–100. Respawn text is left as typed until you step it; a step rewrites it as proper h/m/s with no zero parts (`120s` +1 → `2m1s`, `120s` −1 → `1m59s`, `1m` −1 → `59s`, `1h` −1 → `59m`).
- Rows the server would misread or never spawn are flagged with a **⚠** on the form fields and on the list rows: respawn values it reads differently (`120` with no unit, `1d`, `1.5h`, uppercase units, a space…, with the time it would actually use), group rows (`g` / `ga` / `r`) with Sx and Sy both 0, and vnum 0. Exports report how many remain. See [`docs/mapformat/server-regen.md`](docs/mapformat/server-regen.md).
- Delete rows (row button, or **Del** on the selected spawn), import an existing `.txt`, reset a file, or export the current file in correct 11-column server format.
- Spawns are drawn on the map as markers with their spawn-extent rectangles.
- Optional: **Regen › Load mob_names.txt…** reads a `vnum<TAB>name` list (UTF-8, or a legacy codepage read as Windows-1252). Names then show in a **Name** column of the spawn list, under the Vnum field as you type, on the map next to each mob marker. Spawns whose labels would pile up on the same spot share one instead — `Wild Dog × 300` rather than 300 overlapping names, with different mobs of that spot stacked below each other (up to 5, then `+ N more`) — and that label hangs on one of their markers; spawns that are merely near each other keep their own. The number is how many mobs the rows place (the `count` column added up, so one row with count 56 reads `× 56`). A label turns white when one of its spawns is selected (the selected spawn stays in the number), and **Tools › Mob names on the map** switches the labels off, and in the stats panel. Only mob rows are named (`m` / `ma` / `s`) — for `g` / `r` rows the vnum is a group id. Nothing is stored; **Unload mob names** removes them.
- Optional: **Regen › Load group.txt / group_group.txt…** (pick one or both, or drop them onto the page; load only, nothing is edited). Select a `g` / `ga` row and a label next to its marker on the map shows the pack it spawns — name, leader and members, counted per mob; select an `r` row and the label lists the packs it can pick, each with its chance (from the weights) and contents. The label flips to the other side at the edge of the view and is only drawn for a single selection; the form keeps just one line with the pack name. Mob names come from `mob_names.txt` when loaded, else from the labels in `group.txt`. A pack id or set id missing from the files, and a pack without a leader, are flagged. The **Name** column and the stats panel show pack and set names. The files follow the same rules as mob names: remembered per workspace and recycled for new maps; **Unload group files** drops them.

### Objects (areadata)
Per-sector `areadata.txt` editor:

- Sector dropdown; list of objects with CRC and cell position.
- Edit x/y/z, property CRC, rotation (yaw/pitch/roll) and height bias.
- Add / delete objects (row button, or **Del** on the selected object); markers plotted at (x, −y).
- Click a marker to select it — including one in another sector, which switches the sector — and drag it to move. Dragging is the only way to move an object on the map: clicking the map never relocates one (type X / Y for exact values). **Esc** deselects.
- When a move (drag, stepper or Update) takes an object into another sector, MapForge asks whether to move its record into that sector's `areadata.txt` — **Enter** confirms, **Esc** keeps it where it is.
- Double-click a row to center the map on that object with a mild zoom.
- Selected objects (one or a group) show an arrow for their heading, taken from **Roll**; it turns live as you step or type the value. It uses the same footing as the spawn arrows — 0° south, growing counter-clockwise on the map — which is how the client turns characters but is **not checked in game for objects**, and a model’s own front depends on how it was built, so read it as the rotation. `OBJ_ROT_BASE` / `OBJ_ROT_SIGN` at the top of the script flip it; **Tools › Direction arrows** switches it off.
- X / Y / Z / Bias / Yaw / Pitch / Roll have ▲▼ stepper buttons and respond to **↑ / ↓**: positions step 10 cm (**Shift** 100 cm), rotations 1° (**Shift** 10°) and wrap at 360°. Steps apply live to the selected object; **Enter** in any field runs **Update**, and the button turns gold (`Update *`) while the form has unsaved edits.
- Objects the client would drop or misplace are flagged with a **⚠** on the form and the list rows — CRC 0, a position outside the sector whose `areadata.txt` holds it, fractional rotation — and exports report how many remain. See [`docs/mapformat/areadata-txt.md`](docs/mapformat/areadata-txt.md).
- Export `areadata.txt` (also bundled in the map ZIP).

### Export / import
- **Export map (.zip)** — saved as `<mapname>-YYYYMMDD-HHMMSS.zip` (local time), so every export keeps its own file; the folder inside is still plain `<mapname>`. Re-zips every original file verbatim and rewrites the files you edited (regen/boss/stone/npc, areadata, generated `server_attr`, imported layers). Uses a built-in ZIP writer + CRC32, no external library.
- **Export PNG** — renders the whole current layer to a full-resolution PNG (cols×256 by rows×256).
- **Import PNG** — decodes a full-map PNG back into the current layer's per-sector files. Minimap and tile round-trip losslessly / index-exact; height, shadow and attribute are approximate; water is not supported.
- **Export all PNG** — one full-resolution PNG per layer (`<mapname>_<layer>.png`) in `<mapname>-layers-YYYYMMDD-HHMMSS.zip`; layers without data are left out.
- **Import all PNG** — swap several layers at once. Pick PNG files and/or a `.zip`, pick a folder, or drag and drop any mix onto the page; the layer is read from the file name (`minimap`, `height`, `tile`, `shadow`, `attribute`, also `attr` / `heightmap` / `shadowmap`; with several matches the last word wins, so `<mapname>_<layer>.png` always works). Only the layers found are replaced, a prompt lists them first (layer imports are not covered by undo), images of another size are scaled to the map, and water is skipped. Zips from other tools (deflate) are read too.

### Merge maps
**File › Merge maps…** joins any number of maps into one.

- Load maps with **+ add map** or by dropping their folders anywhere on the dialog (on a grid cell to place them there). Each map keeps its real footprint — one grid cell per sector, so a 6×6 map covers 6×6 cells — and shows a minimap thumbnail. Drag a map anywhere on the board — a fixed 40×40 sheet of cells that scrolls — from the strip or from where it already sits; a placed map keeps the grip you picked it up by, and one dropped near the edge is pulled back inside. Moving is dragging only: a click just highlights a map and its card. While you drag, a ghost the size of the map shows the cells it will land on — snapped, labelled with the target cell, and amber where it would overlap another map. New maps land next to the middle, so there is room to build out in every direction, and the board scrolls to them.
- Maps may overlap. Where they do, the layer order decides which one wins: **↑ / ↓** on a map’s card move it forward or backward, and the overlap is flagged on the card and the grid. Sector files and objects follow the winning layer.
- The board is only working room: there is nothing to resize. The result is trimmed to the area the maps actually cover — empty space on every side is dropped — and a dashed outline plus a line above the board show what you will get (`result: 7×4 sectors · 3 empty sectors will be generated`). Cells inside it that no map covers become real, flat, walkable empty sectors (height, tile, attr, water, minimap and AreaProperty), so the merged map has no holes.
- Every map card shows the textureset file its `setting.txt` asks for (amber while it is missing, green once one is in place; hover for where it was found), so you know which `.txt` to fetch without opening the file.
- Texturesets: each map’s file is taken from a manual attachment (click **+ ts** on its card, or drop the `.txt` onto the card — drop several files or the whole textureset folder and the one named in that map’s `setting.txt` is picked), else from inside the map folder (matched against the `TextureSet` line in `setting.txt`), else from a **YmirWork/textureset** folder you link once in the dialog. They are merged into one, duplicates collapsed, and every `tile.raw` is remapped. `tile.raw` stores one byte per cell, so more than **255** distinct textures cannot work: the merge then stops and lists how many each map contributes, instead of producing wrong ground textures.
- **Setups** (Chrome / Edge): **Merge & preview** saves the arrangement by itself — it updates the setup in use, or keeps one automatic setup (`auto · map_a + map_b`) per combination of maps — and **save setup…** remembers it under a name of your choice — which map folders, where each one sits, the layer order, the linked textureset folder and any textureset attached to a map. It stores references only, like workspaces, and never restores by itself: press **restore** on a setup and the browser asks for read permission once per folder, then everything is re-read from disk (if it wants another click part-way, nothing is half-loaded — press restore again). A folder that was moved is named and the rest still loads. **save** overwrites the setup you restored, **save as new…** adds another, ✎ renames, × removes. Maps added through the plain upload fallback carry no folder reference and cannot be part of a setup.
- Sector numbers, map size, regen coordinates, areadata positions and water layers are all re-adapted. Maps with different CellScale / HeightScale are refused. Regenerate **server_attr** afterwards.

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

- **Highlight all regens** — a chip next to *Highlight attribute* in the layer bar, off by default. It draws the spawns of all four files at once (regen · boss · stone · npc), each in its colour, on every tab, with a small legend and counts in the corner. Display only: selecting, dragging, placing, labels and warnings stay with the current file of the Regen tab, which is drawn on top at full size while the others are a little smaller and see-through.
- **About MapForge** — credits (author and contributors); closes with **Enter** or **Esc**.
- **Direction arrows** (on by default): selected objects get a heading arrow (see Objects), and point spawns with Dir 1–8 get a small arrow showing where they face; Dir 0 (random) and spawns with a range get none, since the server only applies Dir to point spawns. Dir runs counter-clockwise on the map from south — 1 south, 2 south-east, 3 east, 4 north-east, 5 north, 6 north-west, 7 west, 8 south-west — checked against server-side NPC placements.
- **Stats panel**: an overlay in the top-left of the map with rows and summed `count` for every regen file, the current file by type with its warning count and top 10 vnums, and the objects (total, sectors used, distinct and most-used CRCs, warnings). A group row counts its groups, not the mobs inside them.
- **Ruler**: click two points on the map to measure the distance in meters, with Δx / Δy (map units are meters: 1 px of the map image = 1 m). A third click starts a new measurement, **Esc** clears it. While the ruler is on, clicks only measure — nothing is placed or selected — and dragging still pans.

### Workspaces (Chrome / Edge)
MapForge remembers what you opened, so a setup can be reloaded from disk in one click.

- Opening a map folder creates a workspace automatically (named after the folder; opening the same folder again reuses it). Loading a regen folder, dropping spawn files, loading `mob_names.txt` or the group files, or loading a textureset in the paint palette links those to the current workspace. Mob names are recycled: the file you loaded last is what every map without names of its own gets — the next map you open in the same session, and also a brand-new map after a browser restart — and it is saved on that map’s workspace. Loading a different file while a workspace is open switches only that workspace; the others keep theirs, and the new file becomes the one new maps get. If the browser wants a click before it lets the file be read again, the map still opens and **Regen › Reload last mob names** loads them. **Unload mob names** takes the file off the open workspace and stops recycling it.
- The start screen lists your workspaces next to the open box, most recent first, with what each one links. Click one to reload the map, the regens and the mob names from their folders; the browser asks for read permission first (once per session — Chrome can remember it). View settings (layer, tab, tool toggles) come back too.
- ✎ renames a workspace — worth doing, because a web page never sees a full path, so two folders both called `map_a2` look the same until you name them. × removes it from the list; nothing on disk is touched. **File › Workspace … rename** does the same for the open one.
- **File › Export workspaces…** saves the list as `mapforge-workspaces-YYYYMMDD-HHMMSS.json`; **Import workspaces…** adds the entries of such a file (same names get “(imported)” added, nothing is overwritten). A browser cannot write a folder reference into a file, so the export holds names and view settings only: imported entries show as *not linked*. Opening one asks you to pick its map folder once — or simply open a map folder with the same name and it is adopted — and the regens and mob names are linked again the first time you load them for it. Good for moving a setup to another PC or browser; it is not a backup of any map data.
- A workspace stores references to folders and files, never their contents or your edits. If something was moved or deleted it is reported and the rest still loads.
- Needs the File System Access API, so Chrome / Edge (and other Chromium browsers that keep it on, such as Opera and Vivaldi). **Brave** ships with it switched off: enable `brave://flags/#file-system-access-api` and restart. Firefox and Safari cannot hand a page a reusable reference to a real folder at all. Where it is missing, the Workspaces panel says so and everything else works as before. The references live in the browser’s site storage (IndexedDB), which survives restarts but is wiped by “clear site data” and cleanup tools.

### Safety & UI
- **Undo / redo** for every spawn and object edit — place, move, edit, delete, import or reset a regen file, add or delete an object, move an object to another sector. **Ctrl+Z** undoes, **Ctrl+Y** or **Ctrl+Shift+Z** redoes (also as **↶ Undo / ↷ Redo** buttons in the top bar and under the **Edit** menu; both name the next step). Up to 100 steps; a run of stepper presses on one item counts as one step. Undo jumps back to the tab, file and sector where the edit happened. While a form has unsaved typed text, Ctrl+Z is left to the text field. Layer imports, `server_attr` and map merges are not tracked, and loading or closing a map clears the history.
- Confirmation modal on every destructive action (reset a regen file, delete a spawn/object, close map).
- Working menu bar: **File · Edit · Map · Server Attr · Regen · Tools**.
- Resizable inspector sidebar.
- Loads a map folder via the picker **or** by drag-and-drop anywhere onto the page. A dropped folder with a `setting.txt` is a map; loose `regen.txt` / `boss.txt` / `stone.txt` / `npc.txt` files (or a folder holding them) load as spawn files into the current map.

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
