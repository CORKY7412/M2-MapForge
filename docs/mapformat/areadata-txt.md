# `areadata.txt` — Static Object Placement

Per-sectree list of placed objects: buildings, trees, effects, dungeon blocks. Each record references a property file by CRC (see [client-global-refs.md](client-global-refs.md)); the map itself stores only position/rotation/CRC.

- **Path:** `<map>/<XXXYYY>/areadata.txt`
- **Source:** read `CArea::__Load_LoadObject` (`GameLib/Area.cpp:795-890`), write `CAreaAccessor::__SaveObjects` (`WorldEditor/DataCtrl/MapAccessorArea.cpp:1043-1109`)

## Structure

```
AreaDataFile

Start Object000
    <x> <y> <z>            float, world position (cm)
    <CRC>                  uint, property CRC
    <yaw>#<pitch>#<roll>   rotation degrees, '#'-joined (or a bare value = roll only)
    <heightBias>           float (cm), added to z
    [<p0> <p1> ...]        optional portal IDs (bytes)
End Object
...

ObjectCount <N>
```

**Critical parser fact:** the tokenizer (`LoadMultipleTextData`, `EterLib/Util.cpp:51-111`) flattens *all tokens between `Start X` and `End`* into one positional vector — line breaks inside a block are cosmetic. Fields are addressed purely by index:

| Index | Field | Parsed with | Notes |
|---|---|---|---|
| 0, 1, 2 | position x, y, z | `atof` | Map-local world cm. **y is stored negated** (see below) |
| 3 | property CRC | `atoi` | Unregistered CRC → record silently dropped on load |
| 4 | rotation | `atoi` per part | `yaw#pitch#roll` degrees; a value without `#` = roll only (legacy). Written as `%f` but **read with `atoi` → truncated to integer degrees**. Engine parsing defect: yaw is read with `substr(0, s-1)`, dropping the character right before the first `#` (`Area.cpp:850`) — harmless for `%f`-style values (`120.000000` → `120.00000`), but bare-integer style breaks (`90#0#0` → yaw 9). Always write the `%f` form |
| 5 | height bias | `atof` | Final Z = z + bias |
| 6+ | portal IDs | `atoi` each | Up to `PORTAL_ID_MAX_NUM`; used by indoor/portal culling. Line only written when first ID ≠ 0 |

Records with fewer tokens are legal: 4 tokens = position+CRC only, 5 adds rotation, 6 adds bias (this is the implicit "versioning" — old files simply have fewer fields). Both `AreaDataFile` and `ObjectCount` keys are required.

## Coordinates

- Units: **cm**, map-local (relative to the map origin, *not* the sectree origin). A sectree `(ax, ay)` spans `x ∈ [ax*25600, (ax+1)*25600]`, `y ∈ [ay*25600, (ay+1)*25600]` — but stored `y` is **negative**: `stored_y = -(terrain_y)`. Evidence: editor height lookups use `GetTerrainHeight(x, -y)` (`MapAccessorArea.cpp:683`); all real files have negative Y.
- Rotation: yaw/pitch/roll in degrees, applied via `D3DXMatrixRotationYawPitchRoll`. Most objects use only roll (heading around Z).
- Objects are re-numbered `Object000..` contiguously on save; ambience-type objects are excluded (they go to [areaambiencedata-txt.md](areaambiencedata-txt.md)).

## CRC → object resolution

`CPropertyManager::Get(crc)` looks up the property registered from `property/` files (each `.pr*` file stores its own CRC in its body). The property's `PropertyType` (`Tree`/`Building`/`Effect`/`DungeonBlock`) decides the instance type and its `*File` key names the model (`Area.cpp:498-531`). The map never stores model paths.

## Real example (map_a2 `000000`, excerpt)

```
AreaDataFile

Start Object000
    10892.386719 -18171.355469 17784.789062
    26807040
    0.000000#0.000000#120.000000
    -35.000000
End Object
...
ObjectCount 20
```

Object000: position (10,892, −18,171, 17,785) cm, property CRC 26807040, roll 120°, sunk 35 cm into the ground.

## JS parsing

```js
const map = parseStartEndBlocks(text);       // flatten per §structure
const count = parseInt(map.get("objectcount")[0]);
for (let i = 0; i < count; i++) {
  const t = map.get(`object${String(i).padStart(3, "0")}`);
  const obj = {
    x: parseFloat(t[0]), y: parseFloat(t[1]), z: parseFloat(t[2]),
    crc: parseInt(t[3]) >>> 0,
    yaw: 0, pitch: 0, roll: 0, heightBias: 0, portals: [],
  };
  if (t.length > 4) {
    const r = t[4].split("#");
    // Number() keeps fractions; the engine truncates via atoi (see field table)
    if (r.length === 3) [obj.yaw, obj.pitch, obj.roll] = r.map(Number);
    else obj.roll = Number(t[4]);
  }
  if (t.length > 5) obj.heightBias = parseFloat(t[5]);
  if (t.length > 6) obj.portals = t.slice(6).map(Number);
}
```

## What MapForge does

**Warnings.** Every object is checked, without ever blocking an edit or an export:

| Severity | Condition | Why |
|---|---|---|
| Red | CRC is 0 | Not a registered property, so the record is dropped on load (field table, index 3). MapForge cannot see the `property/` tree, so any *other* unregistered CRC goes undetected |
| Amber | Position outside the sector whose `areadata.txt` holds the record | The warning names the sector the position actually lies in (or says it is outside the map, which is also what a positive stored `y` means). Large models placed on a sector edge in real maps can trip this legitimately |
| Amber | Fractional yaw / pitch / roll | The client reads rotation with `atoi`, so `12.5` becomes `12` |

The warning shows under the object form (with the offending fields outlined), as a `⚠` on the affected rows of the object list (hover for the reasons), and as a count in the status bar after exporting an `areadata.txt` or the map ZIP.

**Export.** Fields are addressed purely by position, so a blank or `NaN` would shift everything after it. Every numeric field is therefore forced to a finite number (falling back to 0) and written in the `%f` form, rotation always keeps the `a#b#c` shape — which also sidesteps the yaw `substr` defect — portal IDs are written as integers, and records are renumbered `Object000…` contiguously with a matching `ObjectCount`.

**Editing.** Objects move on the map by dragging their marker only (a click never relocates one). A move changes `x` / `y` only. When a drag, a stepper or Update takes the object into a different sector that exists in the map, MapForge asks whether to move the record into that sector's `areadata.txt` (Enter confirms); on yes it is appended there and the view switches to that sector. Answering no keeps the record where it is — the amber position warning stays — and the question is not repeated until the object has been back inside its own sector. Nothing is asked for a position outside the map or in a sector the map does not have.

## Pitfalls

- Keys are case-insensitive (parser lowercases); `End Object`'s trailing word is ignored — only `End` matters.
- CRCs can repeat across records (same model instanced many times).
- Keep ambience-property CRCs in `areaambiencedata.txt`, not here. The loader doesn't check property types (instance dispatch happens per property, so a misplaced ambience still plays), but an areadata record has no `range` field — the ambience would get range 0 — and the WorldEditor's save path always separates the two files.
- Writing rotation with decimals is fine (engine truncates), but keep the `a#b#c` shape.
- 2D editor displaying objects on a top-down view: plot at `(x, -y)` to match the terrain/minimap orientation.
