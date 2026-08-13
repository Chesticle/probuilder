# Modular Building Mod — Development Roadmap

A kit-of-parts building system for Minecraft: socket-snapping templates, hologram preview,
radial variant selection, user-authored templates, and survival material sourcing from
linked containers.

**Assumption:** solo or very small team, hobby cadence. Phases are sized relative to each
other rather than in calendar time, because that depends entirely on your Java/modding
experience and hours per week. Sizes are S / M / L / XL.

---

## Decide before writing code

These four decisions are expensive to reverse because they change your file format, your
packet schema, or both. Lock them in Phase 0.

| Decision | Recommendation | Why it's irreversible |
|---|---|---|
| **Loader + MC version** | NeoForge, recent 1.21.x. One target only. | Rendering, networking, and keybinds are your three core systems and the three most version-volatile areas in modding. Porting later is a rewrite of the parts you care about. |
| **Material storage** | Palette *roles* (`primary`, `accent`, `trim`, `glass`, `light`, `ground`), resolved at placement. | Storing concrete blocks means N templates × M wood types. Changing later invalidates every saved template. |
| **Template format** | Vanilla `StructureTemplate` NBT for block data + a sidecar JSON for your metadata (sockets, kit, category, slice markers, flags). | You get rotation, mirroring, and block-entity handling for free. A custom format means reimplementing all of it. |
| **Seam convention** | Butt joint or shared column — pick one, enforce kit-wide. | Mixing them produces one-block gaps at corners across your whole library. |

---

## Phase 0 — Skeleton and decisions · **S**

Get a mod that loads and does nothing interesting.

- Gradle project, mod metadata, loads clean on client and dedicated server
- Item registered, keybind registered, a `Screen` that opens and closes
- Network channel with one round-trip test packet
- Data model classes stubbed: `Template`, `Socket`, `Kit`, `Palette`, `PlacementRecord`
- Log-level toggle and a debug HUD overlay you can dump state to (you will use this constantly)

**Exit:** dedicated server + client connect, keybind opens an empty GUI, test packet round-trips.

---

## Phase 1 — Thin vertical slice · **M**

One hardcoded template. Creative mode only. No sockets, no GUI, no materials. The goal is
proving the full pipeline end to end.

- Load a `.nbt` structure from resources into your `Template` object
- Raycast from the player, compute a placement origin, render a translucent hologram
- Bake hologram geometry to a `VertexBuffer`, rebuild only on template / rotation / palette change
- Rotate with a keybind (`BlockState.rotate()` + origin re-normalization for non-square footprints)
- Right-click sends `(templateId, origin, rotation)`; server validates range and places
- Bulk placement with neighbor updates suppressed, single update pass at the end
- `PlacementRecord` written on every placement: before-state, after-state, palette-compressed
- Spatial undo: raycast a block → find record → outline preview → confirm → revert

**Undo revert rule:** walk each position; revert only if the current block still matches what
the template placed. Mismatch means someone changed it — leave it. This one check handles
overlaps, manual edits, mining, and creeper damage.

**Exit:** you can place a wall, rotate it, place another, and undo the first without damaging
the second. This is the single most important milestone in the project.

---

## Phase 2 — Sockets · **L** ⚠️ highest risk

The make-or-break phase. Do not proceed past it until snapping *feels* right.

```
Socket { pos: BlockPos, facing: Direction, types: [String], polarity: A|B }
```

**Mate rule:** `myWorldPos == theirWorldPos + theirFacing`, `myFacing == theirFacing.opposite()`,
`types` intersect, polarity opposes. Four integer comparisons — no tolerances, no epsilon.

**Solve:** find rotation `R` where `R(S.facing) == T.facing.opposite()`, then
`origin = T.pos + T.facing - R(S.pos)`. For UP/DOWN target facings the yaw is unconstrained —
keep the player's current rotation rather than forcing one.

Tasks:

- Socket data on templates, hand-authored for now (two or three test walls)
- Per-chunk index: `BlockPos → placementRecordId`, plus world-space sockets per record
- Candidate query: raycast hit → nearby records → compatible sockets → solve each
- Rank candidates by distance from the ray hit point, pick best
- **Hysteresis** — don't switch candidates until the ray moves a meaningful distance from
  the current one. Without this the hologram flickers and feels broken
- **Cycle key** — step through alternate solutions when auto-pick guesses wrong
- Marker rendering on the chosen socket pair so the player can see what it grabbed
- Toggle to disable snapping entirely (free placement)

**Netcode note:** snapping is UX, not a security boundary. Server syncs nearby placement
records; client solves; client sends the result. Server validates range, protection, and
template existence — it never re-runs the solve.

**Exit:** build a 4-walled room by aiming and clicking, with no fighting the snap. If this
isn't fun yet, stop and iterate — no amount of content fixes bad snap feel.

---

## Phase 3 — Placement modes and terrain · **M**

- **Mode 1 — Snap:** mates to existing placements; terrain ignored
- **Mode 2 — Direct:** snapping off, places on whatever block you're looking at
- **Lock + nudge:** right-click locks the hologram, scroll moves it along the **nearest
  cardinal axis to the camera direction** (quantized — never an arbitrary view vector, or the
  hologram drifts off-grid unrecoverably)
- Second right-click commits; consume the interaction event so locking onto a chest doesn't
  open the chest, and consume scroll while locked so it doesn't swap hotbar slots
- **Replaceability policy:** overwrite blocks in a configurable "natural" tag (dirt, grass,
  stone, sand, gravel, leaves, water); skip everything else
- **Air carve flag** (per template): does template air clear the bounding box, or leave
  existing blocks? Houses usually carve; foundation boxes usually don't
- **Extend-bottom-down flag** (per template): repeat the bottom layer downward per column
  until it hits solid ground. Cheap loop, biggest single win for output quality on slopes

**Exit:** placing into a hillside produces something that looks deliberate.

---

## Phase 4 — Interface · **M**

- Hotkey **tap** → template browser GUI: kits, categories, search, preview thumbnail
- Hotkey **hold** → radial variant wheel (plain / window / doorway / arch / corner)
- **Keybind gotcha:** opening a `Screen` drops keyboard state. Poll raw window key state
  (`InputConstants.isKeyDown` on the window handle) to detect release, or the wheel will
  never close on key-up
- HUD: current template, kit, rotation, mode, snap on/off
- Variant grouping in the data model so the wheel knows what belongs together

**Exit:** you can change template and variant without leaving the world or touching a config.

---

## Phase 5 — Kit content and palettes · **L** (mostly non-code)

This phase is where the promise of the mod actually lives, and it's design work, not
programming. Budget accordingly — the code is maybe 30% of this project.

- `Kit` declares `moduleWidth`, `moduleHeight`, seam convention, socket profiles
- Load-time validation: warn on any template whose dimensions violate its kit's module size
- Radial wheel grouped by kit; cross-kit auto-snap either refused or visually flagged
- Palette definitions (a set of concrete blocks per role) + palette picker UI
- **First kit, authored to spec:** wall (plain, window, doorway), corner, floor, foundation
  box, pillar, and a *deliberately limited* roof set
- Consider recruiting an actual builder. The template library determines whether the mod
  delivers on "good builds without experience," and that is not an engineering problem

**Exit:** one complete, consistent kit that produces a good-looking house.

---

## Phase 6 — Sizing · **M**

Replaces per-cell row/column masking as the primary mechanism.

- **3-slice:** each template marks a start cap, a tileable middle, and an end cap. Longer
  repeats the middle; shorter removes repeats. Windows and beams live in the middle unit, so
  patterns never get cut in half and caps are never mangled
- **Roofs:** 3-slice along the ridge run; author discrete spans (5, 7, 9, 11) as wheel
  variants. Height is a function of span on any pitched roof, so that axis cannot be scaled
- Sockets on caps must survive the middle being repeated — regenerate socket positions from
  the final assembled size, not the authored size
- **Optional:** keep per-cell masking as a power-user extra for prismatic templates
  (foundations, plain walls) where arbitrary trimming is safe

**Exit:** two number fields on the wheel resize a wall; roof span is a variant selection.

---

## Phase 7 — User templates · **M**

- Two-corner selection tool → scan region → save dialog
- Save dialog asks for: name, kit, **category** (dropdown)
- **Sockets auto-derived from category** — this is non-negotiable. If saving requires
  hand-placing sockets, nobody will use the feature

| Category | Generated sockets |
|---|---|
| Wall | `wall_seam` ±X (A/B), `deck` top (A), `deck` bottom (B) |
| Floor | `floor_seam` all four sides, `deck` bottom (B), `deck` top (A) |
| Foundation | `deck` top (A), `floor_seam` four sides |
| Roof | `eave` bottom (B), `roof_seam` on ridge axis (A/B) |
| Pillar | `deck` top and bottom |

- Optional in-game socket editor as a power-user override
- Optional palette abstraction on scan: "which block is `primary`?" mapping UI
- Template library management: rename, delete, export/import as file

**Exit:** a player saves their own wall and it snaps to your kit's walls.

---

## Phase 8 — Survival · **M**

- Material tally computed from template + palette + mask, shown on the HUD
- Inventory check, then linked containers
- **Linked containers:** shift-right-click a chest with the linking item; multiple allowed;
  range-limited and loaded-chunks only
  - Validate range at **use** time, not link time
  - Invalidate a link if the block at that position is no longer the container you linked
  - Use `IItemHandler` (NeoForge) / `Storage<ItemVariant>` (Fabric) — never raw `Container`,
    or modded storage breaks
  - Verify the player can actually open the container before allowing the link
  - Cache the tally; don't rescan containers every frame for the HUD
- Insufficient materials: block placement and show what's missing (do not partial-place)
- Creative bypasses all of it

**Exit:** a full survival build loop with no creative mode.

---

## Phase 9 — Hardening · **M**

- Fire per-block place events so FTB Chunks, GriefPrevention, and friends can veto
- Permission checks per placement; config for max template size and placements per tick
- **Placement record retention:** cap per-player or per-chunk with oldest-evicted, configurable.
  Long-lived worlds otherwise accumulate thousands. Document that evicted = no undo
- **Multiplayer undo:** config toggle for undoing another player's placement, default off
- Custom template sync to server: hash + cache, never resend a template the server already has
- Sodium / Iris compatibility pass — custom world rendering is the #1 source of
  "your mod breaks with shaders" reports
- Performance: large template placement under load, hologram rebuild cost, index memory

**Exit:** survives a real server with 5+ players and a normal modpack.

---

## Phase 10 — Release · **M**

- Second and third kits (different material themes) proving the palette system works
- Config file with sane defaults and comments
- JEI/EMI integration for the linking item if relevant
- Docs: sockets, categories, kit authoring, datapack format
- Wiki page, trailer or GIF set, Modrinth + CurseForge listings

---

## Post-1.0

- Datapack API so others can ship kits without Java
- Community template sharing (format + validation)
- More roof types (hip, gambrel, mansard) as span-variant sets
- Interior kits: furniture arrangements, lighting patterns
- Auto-complete suggestions: highlight the socket where a piece "obviously" goes next

---

## Explicit non-goals for 1.0

Cutting these early keeps scope survivable:

- **Procedural roof generation** over arbitrary footprints — genuinely hard geometry, and
  discrete span variants cover the real use cases
- **Diagonal / 45° builds** — impossible on the block grid, not worth faking
- **Terrain deformation** — the extend-bottom-down flag plus foundation templates covers it
- **Cross-loader support** — pick one, port after 1.0 if the mod finds an audience
- **Curved or organic structures** — different problem, different mod

---

## Risk register

| Risk | Severity | Mitigation |
|---|---|---|
| Snapping feels bad | **Fatal** | Phase 2 exit gate. Hysteresis + cycle key + visible socket markers. Iterate until it's fun before writing any content |
| Template library is mediocre | **Fatal** | It's the entire value proposition. Recruit a builder; enforce module discipline via kit validation |
| Roof scope explosion | High | Hard-limit to 3–4 styles × 4 spans for 1.0. Non-goal the procedural version |
| Shader/Sodium incompatibility | High | Test with Iris from Phase 1, not Phase 9. Retrofitting is much worse |
| Bulk placement lag | Medium | Suppressed neighbor updates + single update pass, benchmarked in Phase 1 |
| Placement record bloat | Medium | Palette compression, per-chunk index, configurable retention cap |
| Protection mod conflicts | Medium | Fire proper events from the start rather than calling `setBlock` blind |
| Version churn mid-development | Medium | Isolate rendering and networking behind interfaces so a port touches few files |

---

## The one-line version

Build the thin vertical slice (Phase 1), then sockets (Phase 2), then stop and honestly
assess whether wall-to-wall snapping is fun. Everything after that is content and polish —
but if Phase 2 doesn't feel good, nothing downstream will save it.
