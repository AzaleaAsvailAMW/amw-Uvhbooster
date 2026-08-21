# Azzy UVH Booster

**Azzy UVH Booster** is a Borderlands 4 SDK mod that puts a full in-game
control panel behind one keybind. It runs on pyunrealsdk / mods_base and draws
its own UMG interface rather than piggybacking on the game's menus.

- **UVH boosting** — rank-up tiers 1–7, max level, spec, gold, eridium, and
  vault cards, applied to any live lobby player
- **BANK** — opens the game's own bank UI on demand via
  `gbx.ui.view.stateadd MENU_BANK`, run through
  `KismetSystemLibrary.ExecuteConsoleCommand` (same call path the engine's
  own console uses)
- **Codes screen** — browse the GZO and Lootlemon catalogs, or list your
  character's real backpack and equipped gear as `@U` serials with names.
  Copy in serial or decoded human-readable form, or grant items directly
- **Movement & rarity** — speed, gravity, jump height, glide, dash and
  traversal costs on live-captured BL4 defaults, plus loot rarity weights,
  with one-click presets
- **Quality of life** — infinite jump, super dash, loot drop/teleport/delete,
  backpack and bank capacity, 27 themes (4 animated), and a movable
  pause-menu button that remembers where you put it

Everything runs on the game thread — no worker threads, no sleeps. Settings
persist across sessions.

**Author:** Azalea Asvail · **Thanks:** Pyrex · **Game:** Borderlands 4
**Stack:** pyunrealsdk / mods_base

Open the panel with the keybind (default **F12**) or the red **AZZY** button in
the pause / title menu.

---

## For anyone (or any AI) working on this file

Read this section before changing code. It is the part that is easy to get wrong.

### Two kinds of screen — do not mix them up

| | Main panel | Fullscreen pages (`codes_ui`, `movement_ui`) |
|---|---|---|
| Built with | `_construct()` | `_construct_in(path, outer)` |
| Lives in | the panel's own `WidgetTree` | **its own** `UserWidget` + `WidgetTree` + `CanvasPanel` |
| Added by | one `AddToViewport` | its own `AddToViewport(Z_BASE)` |
| Coordinates | 1920 × `DESIGN_HEIGHT` design space | 1920×1080 design space |

`_construct()` **always** parents to the main panel's tree. Using it for a
fullscreen page makes that page an invisible child of the panel — it will not
show up as a separate screen in UI editor tools, and its positioning will be
wrong. Use `_construct_in()` for anything with its own tree.

### Coordinate spaces — the single most common bug

Three different spaces are in play:

1. **Design space** — what you author in (1920×1080).
2. **Layout units** — what `CanvasPanelSlot` coordinates actually use.
3. **Raw viewport pixels** — what the mouse reports, and what
   `SetPositionInViewport(..., bRemoveDPIScale=True)` takes.

The conversion is `layout = raw / dpi`. On a handheld at dpi 0.5 a widget
declared 190×92 **draws at 95×46 raw pixels**. So:

* clamping / hit-testing / centring must use `pause_launcher.render_size(dpi)`,
  never the raw `LAUNCHER_W` / `LAUNCHER_H` constants;
* the pause button moves via `SetPositionInViewport(x, y, True)` with children
  slotted at **local (0,0)** — move the *widget*, never the children.

### Threading

Only `gzo_catalog.py` uses a thread, and it imports **zero** unrealsdk — it does
network + JSON only, stores the result behind a lock, and the game thread picks
it up with `poll_refresh_result()`. **Never call into Unreal from a worker
thread.** UObjects are not thread-safe and the engine can destroy one mid-call.

### Session safety

`_invalidate_session_refs()` drops every cached UObject when the player
controller changes (map load, travel, rejoin). This matters because `_live()`
guards by *reading* `obj.Name` — on a destroyed object that read is itself an
access violation, and `except Exception` cannot catch a hardware fault. If you
add a new module-level cache holding UObjects, clear it there.

`_tick_cb` has a `_tick_busy` re-entrancy guard and a tick interval, because the
camera hook can fire more than once per frame.

---

## Fullscreen page architecture

Both `codes_ui.py` and `movement_ui.py` follow the same recipe. Copy either one
to make a third.

**1. Separate module, no circular import.** The page never imports
`__init__.py`. The main mod injects helpers once:

```python
movement_ui.bind(
    construct_in=_construct_in, slot_screen=_slot_screen,
    colour=_colour, vec=_vec, live=_live, get_pc=get_pc,
    set_status=_set_status, themed=_codes_themed, text_tint=_codes_text_tint,
    viewport=lambda: (_viewport_w, _viewport_h, _viewport_dpi),
    **{"try": _try},
)
```

Inside the page, `_h("name")` looks a helper up.

**2. Its own widget.**

```python
uw   = _construct_in("/Script/UMG.UserWidget", pc)
tree = _construct_in("/Script/UMG.WidgetTree", uw)
uw.WidgetTree = tree
root = _construct_in("/Script/UMG.CanvasPanel", tree)
uw.WidgetTree.RootWidget = root
_try(uw, "SetDesiredSizeInViewport", _vec(_layout_w, _layout_h))
_try(uw, "AddToViewport", Z_BASE)
_try(uw, "SetVisibility", 2)          # hidden until opened
```

**3. Resolution-independent scaling.**

```python
_layout_w = raw_w / dpi
_layout_h = raw_h / dpi
_scale = min(_layout_w / 1920, _layout_h / 1080)   # min() preserves aspect
_off_x = (_layout_w - 1920 * _scale) / 2          # centre the design box
_off_y = (_layout_h - 1080 * _scale) / 2

def _px(x, y, w, h):
    return (_off_x + x * _scale, _off_y + y * _scale, w * _scale, h * _scale)
```

Author in 1920×1080; it lands correctly at 720p, 1080p, 1440p, 4K and ultrawide.
`min()` is why ultrawide gets side bars rather than stretching. The full-screen
catcher/backdrop is slotted with `raw=True` (raw `_layout_w/_layout_h`) so it
still covers everything at any aspect.

**4. Buttons.** Coloured `Border` + invisible `Button`
(`SetRenderOpacity(0.02)`) + centred `TextBlock`. Register `(button, handler)`
and poll in `tick()`:

```python
pressed = btn.IsPressed()
if was and not pressed:
    handler()          # fire on release
```

Poll **each button's own `IsPressed()` only**. Never OR in a global mouse-down
flag — a click anywhere then fires whichever button is first in the list.

**5. Input ownership.** `_poll_ui_buttons()` returns early while a page is open:

```python
if codes_ui.is_open() or movement_ui.is_open():
    return
```

Without this, one click hits the page *and* the panel underneath.

**6. Theming.** Colours route through `_h("themed")` → `_codes_themed()`, which
resolves animated themes (Rainbow, GZO, Cow, Beveyism, Darling) to a current
colour. Plain `_themed()` returns those unchanged because the panel animates
them through its own widget list, which the pages are not part of.
`_apply_theme_now()` calls each page's `on_theme_changed()`.

### Wiring a new page into `__init__.py`

Eight places, all mirroring `codes_ui`:

1. `from . import <page>_ui`
2. `_bind_<page>_ui()` — the `bind(...)` call
3. `open_<page>_ui()` / `close_<page>_ui()`
4. a `_button(...)` on the panel that calls `open_<page>_ui`
5. `<page>_ui.build()` next to `codes_ui.build()`
6. `<page>_ui.tick()` in `_tick_cb`
7. `<page>_ui.on_theme_changed()` in `_apply_theme_now()`
8. `<page>_ui.destroy()` in the disable path, and add it to the
   `_poll_ui_buttons()` input gate

Give the new page a distinct `Z_BASE` so pages never fight over z-order.

---

## The Movement UI

`movement_ui.py` — fullscreen, paged, themed, scales like the Codes UI.

**To add a new movement setting you do not touch the UI at all.** Add one row to
`movement.SPECS`:

```python
("key", "Label", low, high, step, "what it does")
```

and its default to `movement.VANILLA`. The page count grows on its own — 6 rows
per page, so 9 settings = 2 pages, 20 settings = 4 pages. `<` / `>` and the draggable
page slider mirror the Codes UI paging pattern.

The actual field writing lives in `movement.py`, which knows the real BL4 field
names (`MinAnalogWalkSpeed`, the nested `JumpGoal.GoalHeight`, and the
`VaultPowerCost_*.Value` struct paths). Run `azzymove fields` in-game to see
which of them a given build actually exposes.

| Button | Does |
|---|---|
| APPLY TO SELECTED | writes the values to the players selected in the panel |
| RESET TO VANILLA | restores defaults and re-reads the sliders |
| REFRESH | re-syncs the visible sliders and values |
| < PREV / NEXT > | page through the settings; the page slider can also be dragged |
| CLOSE | hide the screen |

Sliders write straight back into `_move_values` through the `set_value` helper,
which also updates the saved mod-menu option so the value persists.

---

## The Codes UI

`codes_ui.py` — fullscreen list of BL4 codes with filters, search, paging and
multi-select. `gzo_catalog.py` fetches the catalogue on a worker thread;
`REFRESH` triggers it and the game thread collects the result via
`poll_refresh_result()`. `lootlemon_catalog.py` supplies the bundled list.

---

## Pause-menu AZZY button

Its own `UserWidget` at a high z-order, shown via `MenuOpen` / `MenuClose` hooks
on `/Game/UI/Scripts/ui_script_menu_base...`. Never built from inside a menu
hook — construction is deferred onto the camera tick via
`_schedule_launcher_umg()`, because building during a menu transition crashed
when other mods were loading.

Position is stored in three places kept in sync by
`_persist_launcher_position()`: the `LauncherX/YOption` sliders, a
`launcher_position.json` sidecar, and a soft-sync into `user_settings.json`.

Two rules that are easy to break:

* **Nothing may write a `(-1, -1)` pair to the sidecar except a real reset.**
  `mods_base` assigns option defaults of −1 while loading, which fires
  `on_change` *before* `on_enable` reads the sidecar; without the guard the good
  saved position is destroyed on every launch.
* **`_sanitize_launcher_position()` must not persist its clamp.**
  `_launcher_position()` already clamps on read for display. Writing the clamp
  back means any boot where the viewport differs (title screen vs in-game)
  permanently nudges the saved spot, drifting further every restart.

`azzybutton` in console prints widget/menu/force state, the saved coords, the
viewport, dpi and remaining defer time. Use it before changing anything here.

---

## Settings that persist

Theme · panel position · window scale · super dash strength · movement values ·
AZZY button position · inventory capacity.

Setting `Option.value` only changes memory — `mod.save_settings()` writes to
disk. `_save_settings()` wraps that and must be called after any option write.

---

## Console commands

| Command | Does |
|---|---|
| `azzytheme <name\|list>` | set / list themes |
| `azzyscale <value>` | panel size |
| `azzydashstrength <n>` | super dash strength |
| `azzymove list\|set\|apply\|reset\|fields` | movement settings |
| `azzybutton status\|show\|hide\|reset\|arm\|probe` | pause-button diagnostics |
| `azzy_bg <path>` / `azzy_bg_clear` / `azzy_bg_opacity <0-1>` | panel backdrop |

---

## Files

| File | Purpose |
|---|---|
| `__init__.py` | main mod: panel, keybinds, options, hooks |
| `codes_ui.py` | fullscreen codes page |
| `movement_ui.py` | fullscreen movement page |
| `movement.py` | movement field tables + apply/reset |
| `gzo_catalog.py` | threaded code-catalogue fetch (no unrealsdk) |
| `lootlemon_catalog.py` | bundled code list |
| `inventory_capacity.py` | backpack / bank size |
| `live_items.py` | reads live backpack / equipped gear into code rows |
| `pause_launcher.py` | AZZY button geometry, dpi, menu-name matching |
| `serial_convert.py` | Base85 ↔ readable serials |
| `themes.json` | theme palettes |
| `assets/` | theme backdrops |


### Movement UI v1.80

- Movement sliders use the same dark-track / blue-fill / white-handle treatment as Super Dash and Backpack/Bank.
- Removed the duplicate `REFRESH PAGE`; the large bottom `REFRESH` button is now the single refresh action and reloads saved movement options before redrawing.
- Added one-click presets: `VANILLA`, `GLIDE`, `FAST`, and `FLOAT`.
- Gravity minimum is now `0.01` for very floaty/glide-heavy setups.

## Movement & Rarity

The fullscreen Movement screen now combines movement tuning and loot rarity
weights into the same Codes-style paged UI. Movement uses the live-captured BL4
vanilla baseline, while rarity weights are 0-100% sliders backed by the current
GameState.RarityState path. Rarity intent is cached separately and re-applied in
a short burst after world/GameState changes instead of being rewritten every
frame. The page list grows automatically as movement or rarity rows are added.

## Numeric contract (BL4 live-captured values)

Movement uses the live-captured defaults and slider ranges from MattsSDKBoostingTools:

- Speed Scale: 0.05-25.0, step 0.01, default 1.0x
- Walk / Ground Speed: 50-10000, step 1, default 600
- Master JumpGoal Height: 0-10000, step 1, default 198
- SprintJump GoalHeight: 0-10000, step 1, default 198
- DoubleJump GoalHeight: 0-10000, step 1, default 225
- SlideJump GoalHeight: 0-10000, step 1, default 198
- Gravity Scale: 0-10, step 0.01, default 1.00
- Max Step Height: 0-1000, step 1, default 45
- Walkable Floor Angle: 0-89.9, step 0.1, default 44.76508331298828
- Walkable Floor Z: 0-1, step 0.001, default 0.7099999785423279
- Gliding Speed: 0-30000, step 1, default 1200
- Gliding Speed Boost: 0-30000, step 1, default 0
- Gliding Air Control: 0-50, step 0.01, default 0.6000000238418579
- Dash Speed: 0-30000, step 1, default 2500
- Time Dilation: 0.01-64, step 0.01, default 1.0x

Non-slider movement defaults remain jump_count=2, jump_off_z_factor=0.5 and
jump_hold_time=0.0. Individual jump goals default to off. Vault power costs are
zeroed on apply by default, while RESET restores the captured vanilla state.

Rarity uses 0-100% UI values, 0.0001 change threshold and 0.001 live-match
tolerance. World-change reapply checks run on a 0.25s schedule with 0.75s
retries inside a 12s burst; the burst stops as soon as the live GameState matches.


## Infinite Dash

The Movement & Rarity screen includes **Infinite Dash Strength**, wired to the existing Super Dash keybind. The slider uses the same 100-20,000 range and 250 step as the main Super Dash control. Presets carry the complete dash-strength value so preset order cannot leave the previous preset's dash strength behind.


## Native Glide / Dash / Vault
Movement UI uses the game-native DashSpeed system, not Super Dash. Dash Speed writes all six native dash fields. Glide writes support fields and captured accel/decel formula. Vault 0 uses struct-first GbxAttributeFloat writes and is host/standalone guarded.


## INV / EQP filters (Codes UI)

Two extra filters on the Codes screen list the character's **real** gear as
serial + name instead of catalog entries:

* **INV** — unequipped backpack items
* **EQP** — equipped items, labelled by slot

Both read `PlayerState.BackpackItems`. That is a FastArraySerializer
**struct**, not an array — iterating it directly raises "object is not
iterable". The rows live in an inner field, lowercase `items` on this build
(the probe dump shows `EquippedInventorySlots` fields as
`['ArrayReplicationKey', 'DeltaFlags', 'items']`). `_rows_from_container()`
tries the known field names, then falls back to scanning for any array whose
first row looks like an item slot, and logs which field it used.

Note `GbxItemContainerSlot` has **no** `EquipSlot` — only the `InventoryItem`
inside it does. `Flags` carries `EInventoryItemFlags.Equipped` (bit 0) as a
cross-check when the field is missing.

Equipped gear is **not** a separate container: it is the same rows with `EquipSlot` in `0..64` (0-3 weapons, 4
shield, 5 ordnance, 6 repkit, 7 enhancement, 8 class mod). Unequipped rows are
`EquipSlot = -1`. Schema confirmed by Matt's `TwitchInteractionProbe` dump,
2026-08-10.

Live rows are kept in `codes_ui._live_rows`, deliberately **outside** `_codes`,
so ALL / LEGIT / MODDED / LOOTLEMON never show a 999-row backpack. Selection,
COPY and GIVE resolve against `_all_rows()` (catalog + live), so you can select
gear under EQP, switch back to ALL and still copy it.

Clicking INV or EQP always re-scans — cached rows would hand GIVE a serial for
an item the player has since dropped.

### Reading the serial — the risky part

The `@U` serial is not a UProperty. It sits in a `std::string` on the
`InventoryIdentity` struct and is read by offset, the same layout MSBT's
`item_serial_reader.py` uses:

| Offset | Meaning |
|---|---|
| `+0xA0` | string buffer — inline when capacity < 16, otherwise a pointer |
| `+0xB0` | length |
| `+0xB8` | capacity |
| `+0xC4` | item level |

`live_items._read_identity_fields()` validates `0 < length <= capacity <= 8192`
and range-checks the pointer **before** dereferencing anything, because a
ctypes fault is a hardware fault that `except` cannot catch. If a BL4 patch
moves these offsets, `live_items.disable()` is the kill switch and
`live_items.is_disabled()` reports it.

### Names

A serial carries no display name, only balance/part indices. Names resolve in
this order:

1. **Exact serial match** against the loaded catalog — gives the real name.
2. **Fingerprint match** — leading varint + 6th numeric, unique for every
   distinct name across all 330 bundled codes.
3. **Category from the leading varint.** That value partitions the catalog
   cleanly: 70 distinct values, zero collisions across Weapons / Shields /
   Class Mods / Enhancements / Ordnance / Repkits. Most real loot is a random
   roll that is in no catalog, so this is the case that fires most often — a
   non-catalog gun reads `Weapons | Lv 60` rather than "Unknown item".
4. Failing all of that, the equip-slot label, then plain `Item`.

The level comes from the memory read, falling back to the 4th numeric in the
serial when that read comes up empty.

Resolution runs only for the rows drawn on the visible page, so a full backpack
never costs 999 bit-level decodes on the game thread. Widening the GZO catalog
(press REFRESH on a catalog filter) widens both the name and the category map.

## HUMAN toggle

The `HUMAN` button on the Codes action row switches both the list and
`COPY SELECTED` to the decoded readable form produced by
`serial_convert.serial_to_human()` — identical output to the standalone
`Serial_Converter.py`, so copied text pastes straight into the editor box. The
code column widens while it is on, decodes are cached, and only the visible
page is ever decoded.

Round-tripping readable text back through `human_to_serial()` reproduces the
original serial exactly for 329 of the 330 bundled codes; the one exception
differs only in trailing base85 padding, with identical decoded content. That
is existing converter behaviour, not something the UI introduces.
