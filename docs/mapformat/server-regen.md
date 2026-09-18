# `regen.txt` / `npc.txt` / `boss.txt` / `stone.txt` — Server Spawn Files

Server-side spawn definitions, all four sharing one grammar (`regen_load`, `m2dev-server-src/src/game/regen.cpp:684-797`). Conventionally: `regen.txt` = monsters, `npc.txt` = NPCs, `boss.txt` = bosses, `stone.txt` = metin stones — but the parser is identical; the split is organizational.

- **Path:** `<server map dir>/regen.txt` (etc.), loaded per map at boot
- **Encoding:** plain text, whitespace/tab-separated tokens, `"quoted"` tokens allowed, `//` token starts a comment through end of line

## Line grammar

Canonical header comment (written by tools, ignored by parser):

```
//type	cx	cy	sx	sy	z	dir	time	percent	count	vnum
```

| # | Column | Type | Meaning |
|---|---|---|---|
| 0 | `type` | letter(s) | Spawn kind — see below. Only the first char (plus `a` for `ga`) matters |
| 1 | `cx` | int | Rect center X, map-local units of 100 (i.e. meters) |
| 2 | `cy` | int | Rect center Y, same units |
| 3 | `sx` | int | **Half-range** in X (radius, not a start coord, despite the name) |
| 4 | `sy` | int | **Half-range** in Y |
| 5 | `z` | byte | Z-section — passed only through the point-spawn path (`SpawnMob` Z); ranged/group/anywhere spawns ignore it. 0 = ground |
| 6 | `dir` | byte | Facing: `0` = random (one of 8×45°); `1..8` → angle `(dir-1) * 45°`. On the map this runs counter-clockwise from south: `1` S, `2` SE, `3` E, `4` NE, `5` N, `6` NW, `7` W, `8` SW (checked against server-side NPC placements, not read from source). Very old client-side regen files used a flipped direction convention, so do not take those as a reference — the server mapping here is the one that counts. Only applied to point spawns |
| 7 | `time` | duration | Respawn interval: digits + `s`/`m`/`h` suffixes, additive (`1h30m` = 5400 s). Digits **without a suffix are discarded** (`regen.cpp:187-217`), and `time == 0` disables the line completely — not even the initial spawn happens (`regen.cpp:766`). Always write a suffix |
| 8 | `percent` | int | **Parsed and discarded** — the server ignores this column entirely (`regen.cpp:222-224`) |
| 9 | `count` | int | Max simultaneous spawns from this line |
| 10 | `vnum` | int | Mob vnum (`m`), group id from `group.txt` (`g`/`ga`), or group-group id from `group_group.txt` (`r`) |

### Type letters (`regen.cpp:107-130`)

| Token | Type | vnum meaning | Notes |
|---|---|---|---|
| `m` | MOB | mob_proto vnum | `m1`, `m2`… also parse as `m` (trailing chars ignored) |
| `g` | GROUP | group id | spawns the whole group from `group.txt` — **requires nonzero extents** (see below) |
| `ga` | GROUP aggressive | group id | same + `is_aggressive = true` |
| `r` | GROUP_GROUP | group-group id | picks a weighted random group from `group_group.txt` — same extent caveat |
| `s` | ANYWHERE | mob vnum | random position anywhere on the map; rect ignored |
| `e` | EXCEPTION | — | no-spawn rectangle; line ends after column 5 (`z`) — no dir/time/percent/count/vnum. **Dead in stock source**: the only `is_regen_exception` check is commented out (`char_manager.cpp:453-461`), so `e` rows are parsed and stored but have no effect |

**Point-spawn trap:** the spawn dispatcher checks `sx==ex && sy==ey` *before* the type switch (`regen.cpp:417`). A `g`/`ga`/`r` line with `0 0` extents takes the point path, which calls `SpawnMob` with the *group id as if it were a mob vnum* — the group never spawns. Give group lines a nonzero range.

Any other letter = fatal error (server exits).

## Coordinate math (`regen.cpp:142-170, 707-742`)

```
sx_world = (cx - sxHalf) * 100 + BasePosition.x
ex_world = (cx + sxHalf) * 100 + BasePosition.x
sy_world = (cy - syHalf) * 100 + BasePosition.y
ey_world = (cy + syHalf) * 100 + BasePosition.y
```

- Columns are in **1/100 world units** (= meters); the parser multiplies by 100. Equivalently: one regen unit = one client attr half-cell, so valid `cx` ∈ [0, MapSizeX·256] — community tools (Map-Converter) overlay regen points directly on the 256/sectree attribute grid.
- `BasePosition` comes from the map's `Setting.txt`.
- If `sxHalf == 0 && syHalf == 0` the rect degenerates to a point → exact-position spawn (used for NPCs).
- Exception (`e`) rects get the ×100 scaling but **not** the base offset (they're compared map-locally).

### Worked example (`metin2_map_n_desert_01`, base 204800×486400)

```
g	783	205	491	113	0	0	1m	100	174	62
```
→ GROUP id 62, respawn 60 s, up to 174 groups, world rect X [29,200 + 204,800 = 234,000 … 332,200], Y [495,600 … 518,200].

```
m	109	1427	0	0	0	0	1m	100	1	10016
```
→ single mob vnum 10016 at exact point (10,900, 142,700) + base.

## `time` column — grammar, traps and safe bounds

### How the server reads it (`regen.cpp:187-217`)

The word is scanned one character at a time with a running number `tmp` and a total `time`, both starting at 0:

| Char | Effect |
|---|---|
| `0`–`9` | `tmp = tmp * 10 + digit` |
| `h` | `time += tmp * 3600`, `tmp = 0` |
| `m` | `time += tmp * 60`, `tmp = 0` |
| `s` | `time += tmp`, `tmp = 0` |
| anything else | skipped — and `tmp` is **not** reset |

At the end of the word, whatever is left in `tmp` is thrown away. Nothing is ever rejected or logged. From that:

- Only lowercase `h`, `m`, `s` are units. No `d`, no `w`, no uppercase.
- Digits only count when a unit letter follows them; a bare or trailing number is dropped.
- Units add up, may repeat and may come in any order (`5s1h` = 3605, `1m1m` = 120). There is no range check per unit (`90m`, `120s`, `48h` are fine), and zero parts are harmless (`1h0s` = 3600).
- Because unknown characters don't reset `tmp`, the digits on both sides of them glue together.

### What the value means

The interval is a periodic **refill tick per line**, not a per-mob respawn delay: every `time` seconds the line spawns `count − alive` mobs. The first tick fires at `time` plus a random 0–16 s, later ticks exactly every `time` seconds.

`time == 0` on a map file (`regen.txt` / `npc.txt` / `boss.txt` / `stone.txt`) skips the line completely — the mob is never spawned, not even once (an NPC's position is still registered for the minimap). Dungeon regens loaded through `regen_do` behave differently (spawn once, a single refill a few seconds later, then the event ends).

### Safe grammar

```
time   := ( NUMBER UNIT )+
UNIT   := 'h' | 'm' | 's'        lowercase only
NUMBER := decimal digits, no sign, no dot
```

Always end on a unit letter. For days use hours (`24h`, `120h`, `720h`).

### Safe bounds

| Bound | Value | Why |
|---|---|---|
| Minimum | `1s` | Legal, but it means one refill pass per second for that line — avoid on busy maps |
| Practical maximum | `20000h` (~833 days) | Comfortably below the overflow ceiling; anything up to a few weeks is fully defined |
| Hard ceiling | ≈ 85,899,000 s (≈ 23,860 h, ≈ 994 days) | On the stock 32-bit build the interval is multiplied by 25 pulses/s and stored in a signed 32-bit event key. Past the ceiling the key goes negative: the event fires once at boot and then dies, so mobs spawn once and never respawn. Server uptime is added to the key, so the ceiling shrinks the longer the server runs, and it drops proportionally if `passes_per_sec` is raised |
| Wrap | > 171,798,691 s | The unsigned multiply wraps and yields a short, wrong, positive timer |
| Digit run | > 2,147,483,647 | Overflows the `int` accumulator (undefined behaviour); same for more than 596,523 before `h` or 35,791,394 before `m` |
| Token length | < 256 chars | The tokenizer copies each word into a 256-byte stack buffer without a bounds check — this applies to every column, not only `time` |

### Trap inputs

| Written | Server reads | Why |
|---|---|---|
| `120` | 0 → never spawns | No unit, number dropped. The most dangerous one: it looks valid |
| `1h30` | 3600 | Trailing `30` dropped |
| `1d`, `5d`, `30d` | 0 → never spawns | `d` is skipped and the number is never flushed |
| `1d12h` | 403,200 (112 h) | `d` skipped, digits glue into `112` |
| `1.5h` | 54,000 (15 h) | `.` skipped, digits glue into `15` |
| `-1s` | 1 | `-` skipped: a refill every second, not "never" |
| `2m-1s` | 121 | `-` skipped, so it *adds* a second |
| `1H`, `5M`, `10S` | 0 → never spawns | Uppercase is not a unit |
| `s30` | 0 → never spawns | The unit flushes 0, then `30` is dropped |
| `30ms` | 1800 | `m` flushes 30 minutes, `s` adds 0. Not milliseconds |
| `1e3s`, `0x10s` | 13, 10 | Letters skipped, digits glue |
| `10sec`, `10min`, `1hour` | 10, 600, 3600 | Right only by accident — the extra letters are skipped |
| `1h 30m` (unquoted) | 3600 + column shift | The space ends the word: `30m` lands in `percent`, `percent` in `count`, `count` in `vnum`. The whole line is misread, silently |
| `"1h 30m"` (quoted) | 5400 | Quoted tokens may contain spaces; the space is then just a skipped char |
| empty / missing | column shift | The next word is consumed as the time |

### What MapForge does

The Respawn field accepts free text and exports it as typed, so a trap value stays a trap until it is edited — but it is flagged. MapForge runs the same character loop as the server on every respawn value and warns, without ever blocking, when the result is not what the text looks like:

- **Red** — the line would never spawn or is corrupted: the server reads 0 (`120`, `1d`, `1H`, empty, `0s`), or the value is past the overflow ceiling.
- **Amber** — the server reads a time, but not the one written (`1h30` → 1h, `1.5h` → 15h, `2m-1s` → 2m1s, `30ms` → 30m), the value is legal but risky (`1s`, above `20000h`), or it contains a space (which would split the row — MapForge removes it on export).

Two more row-level checks use the same warning, both red:

- a `g` / `ga` / `r` row with `sx` and `sy` both 0 — the [point-spawn trap](#type-letters-regencpp107-130): the group id is read as a mob vnum and the group never spawns;
- `vnum` 0 — nothing to spawn.

`ma` / `ra` are deliberately not flagged: stock source degrades them to `m` / `r`, but some server sources implement them. A `count` of 0 cannot occur in MapForge — import and the form both fall back to 1.

The warning shows on the offending form fields (coloured border; the Respawn field also gets a `⚠`) with a line saying what the server reads, as a `⚠` on the affected rows of the spawn list (hover for the reason), and as a count in the status bar after exporting a regen file or the map ZIP.

The ▲▼ / ↑↓ stepper only ever writes the safe grammar: it reads the current value (treating a bare number as seconds and ignoring case), steps it, and rewrites it as `h`/`m`/`s` parts with no zero parts, never going below `1s` — so stepping a bare `120` once turns it into a valid tag.

## File structure — word stream, line endings, and what stops the server booting

The parser is a **word stream, not line based**. Newline, carriage return, space and tab are all the same separator, and a row ends when its 11th word (`vnum`) has been read — not at the end of the line (`e` rows end after their 6th word). The only place a newline matters is ending a `//` comment.

### Line endings and the last row

| File ends with | Result |
|---|---|
| `…101\n` | Row read once, then a clean end of file |
| `…101\r\n` | Same — `\r` is a separator, so the vnum is a clean `101` |
| `…101` (no end of line) | Same — the pending word is still returned at end of file |
| blank lines, trailing spaces or tabs | Same — leading whitespace is skipped |
| a last `// comment` with no newline | Fine |

So the last row is never lost, duplicated or turned into a phantom row, whatever the ending. LF and CRLF are both safe.

### What does break

| Problem | What the server does |
|---|---|
| Last row truncated (fewer than 11 words) | Dropped silently, no log |
| A row with a missing column anywhere | The stream shifts: the next row's type letter is consumed as this row's vnum, then a number arrives where a type is expected → `unknown regen type` → **the core exits, the server does not boot** |
| An extra word on a row (12 columns, a space inside a value, a comment without `//`) | Same shift, usually the same exit |
| Comment glued to the vnum (`101//dog`) | It is one word: the vnum reads as 101 but the comment is never recognised, so the rest of that line is parsed as the next row. `101 // dog` (with a space) is fine |
| UTF-8 BOM at the start of the file | The first word starts with `EF BB BF`, which is an unknown type → exit. Save without a BOM |
| Stray byte at end of file (Ctrl-Z `0x1A` from a DOS editor) | A one-character word with an unknown type → exit on FreeBSD. Windows builds treat `0x1A` as end of file in text mode, which hides it |
| CR-only (old Mac) line endings | Rows parse, but a `//` comment swallows everything up to the next `\n` — i.e. the rest of the file |
| Extra words after an `e` row's 6th word | They start a new row |

### What MapForge does

- **Export** writes LF endings, no BOM, a trailing newline, and exactly 11 words per row (6 for `e`). Every column is forced to a single non-empty word: whitespace and `//` are stripped from `type`, `time` and `percent`, an empty value falls back to a safe default (`m`, `1m`, `100`), and numeric columns are written as integers. A row can therefore never shift the ones after it.
- **Import** is line based and more forgiving than the server: it strips `//` comments even when glued to a value, tolerates a BOM and CRLF, and fills missing trailing columns with defaults. Rows whose type does not start with `m`, `g`, `r`, `s` or `e` — which would stop the server booting, and which is also what a stray `0x1A` looks like — are skipped, and the status bar reports how many.
- Importing a file and exporting it again is a quick way to clean up a BOM, glued comments, stray bytes and odd line endings.

## Referenced global files

- **`group.txt`** (`mob_manager.cpp:309-378`) — `CTextFileLoader` brace format:
  ```
  Group  Name
  {
      Vnum    101
      Leader  "label"  2001      ← 2nd token = leader mob vnum
      1       "label"  2002      ← numbered members 1..255, 2nd token = mob vnum
      2       "label"  2002
  }
  ```
- **`group_group.txt`** (`mob_manager.cpp:248-307`) — same shape; numbered rows are `<groupVnum> [probabilityWeight]`.

## WorldEditor's regen dialect (compatibility note)

The WorldEditor reads/writes a map-root `regen.txt` with the same 11 columns but only understands types `m` and `g`, requires all 11 tokens, and always writes `z=0`, `time=1m`, `percent=100` (`MapAccessorOutdoor.cpp:1620-1655`). It also emits `MonsterArrange.txt` — a deduplicated list of used vnums, one per line, read by nothing in the engine (content-pipeline aid). An editor supporting full server regen syntax (all 6 types, `e` short rows, time suffixes) is a superset of the WorldEditor dialect.

## Pitfalls

- `sx`/`sy` column names collide with "start x/y" intuition — they are **half-extents** around the center.
- `percent` does nothing; don't surface it as meaningful (keep writing `100` for byte-compat).
- `e` lines are shorter (6 tokens); a column-count-strict parser must special-case them.
- `time` without a suffix silently zeroes the interval and kills the line (see column table) — normalize to `Ns`/`Nm`/`Nh` on save.
- The parser only checks `token[0][0]`, so junk like `mob` parses as `m` — don't rely on it when writing. Community tools emit `ma`/`ra` as "aggressive" variants, but the server only honors the `a` suffix on `ga` — `ma`/`ra` silently degrade to plain `m`/`r`.
- `time` accepts mixed suffixes in any order, but **not** bare digits, days, uppercase, signs or decimals — see [the `time` column section](#time-column--grammar-traps-and-safe-bounds) for the trap table and safe bounds (`1m` style is conventional).
