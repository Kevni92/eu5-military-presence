# EU5 Military Presence

Experimental Europa Universalis V mod that adds a scripted **Military Presence** system for land provinces and a virtual army patrol assignment.

## MVP scope

Every non-exiled army generates Military Presence automatically in the fully owned province where it is physically stationed. No patrol route or player action is required for this baseline effect.

Patrols are modeled **virtually**. Armies do not receive physical movement orders and their map position is not changed.

A patrol army maintains an ordered, circular route of provinces. While the army is assigned to Military Patrol duty, it contributes Military Presence to its current virtual province. Once that province reaches the configured threshold (90), the army advances to the next province in its route. After the last province it wraps back to the first.

Military Patrol is not implemented as a `unit_ability`. The assignment is stored as scripted state on the army (`mp_patrol_active`). An army with an active patrol contributes through its virtual patrol route instead of simultaneously contributing at its physical location, preventing double counting of the same army.

### Using the system in game

No configuration is required for normal stationed armies:

1. Station one of your armies in a province that you fully own.
2. Let a monthly pulse pass.
3. Open the **Military Presence** map mode in the Military map-mode category.
4. The province should gain Military Presence each month while the army remains there.

### Army right-click patrol controls

Right-click one of your own armies to open the normal EU5 unit context menu. Military Presence adds its patrol controls above the vanilla quick unit actions.

- **Edit Military Patrol Route** is visible while the army is not actively patrolling.
- **Start Military Patrol** is visible only when the army has a committed route and is not already patrolling.
- **Stop Military Patrol** is visible only while that army is actively patrolling.

### Multi-province route editor

**Edit Military Patrol Route** opens one province-selection session for the selected army. The editor keeps a temporary working list (`mp_patrol_edit_provinces`) and does not alter the committed route until **Apply** is pressed.

Available selection methods:

- click a fully owned province on the map to add that single province;
- click a province checkbox to toggle that province on/off;
- **Shift + checkbox** adds every fully owned province in that province's Region;
- **Ctrl + checkbox** adds every fully owned province in that province's Area;
- **Select All** adds every fully owned province in the country;
- **Clear Selection** empties only the working copy.

Selected provinces are shown in the selection map with a green base and gold stripes. Map clicks are additive. Removal is explicit through the checkboxes or Clear Selection. The Generic Action map callback does not expose a verified Shift/Ctrl modifier scope, so Region/Area modifier selection is intentionally attached to the checkbox UI rather than map clicks.

- **Apply** replaces the live patrol route with the working list and rebuilds the circular linked list from scratch.
- **Cancel** discards the working list and leaves the live route unchanged.
- Applying an empty working list deletes the committed route.

Retained provinces preserve their relative order. Newly selected provinces are appended in selection order. Removing and then re-adding a province therefore moves it to the end of the route.

Patrol routes cannot be edited while the patrol assignment is active. Stop the patrol first, edit the route, then start it again.

## Military Presence map mode

A custom **Military Presence** map mode is defined under `in_game/gfx/map/map_modes/` in the Military category.

Military Presence remains authoritative at province scope. Because EU5 map-mode coloring is evaluated per location, the current province value is mirrored to all locations in that province as `mp_military_presence_map_value`. This makes the whole province display the same base color.

- no presence: grey
- low presence: red
- high presence: green
- **gold stripes:** at least one army is currently assigned to build Military Presence in that province
- tooltip: current value from 0 to 100 plus assignment state

The assignment overlay includes both unassigned armies at their physical location and active patrol armies at their current virtual patrol target. Active patrol armies in combat do not contribute and are not striped for that pulse. The cache is rebuilt after each monthly Presence update and immediately when a patrol is started or stopped. Physical army movement therefore updates the stripe on the next monthly refresh.

## Current balance

- Military Presence range: `0..100`
- Patrol switch threshold: `90`
- Stationed-army contribution: `+12` per army per month
- Active patrol contribution: `+12` per army per month
- Passive decay: `-2` per month
- At 100 Military Presence, the scaled province modifier provides:
  - `-0.10` local unrest
  - `+0.01` local monthly control
  - `+0.10` local maximum control

Decay is processed before army contributions. A province already carrying Military Presence therefore gains a net `+10` in a month with one contributing army (`-2 +12`). Multiple unassigned armies physically stationed in the same fully owned province currently stack their fixed contributions.

The province modifier scales linearly with Military Presence. Its static modifier is reapplied with an explicit `size = Military Presence / 100`, so refreshes are absolute rather than cumulative. Therefore 22 Presence means 22% of the static modifier (+2.2% maximum Control, +0.22% monthly Control, -2.2% unrest); 50 Presence gives +5% maximum Control and 100 Presence gives +10% maximum Control.

## Technical design

Per-province state:

- `mp_military_presence` — authoritative numeric Military Presence value;
- `mp_military_presence_assignment` — display-cache flag while an army is assigned to build Presence there.

Per-location display cache:

- `mp_military_presence_map_value` — mirrored Presence value used by the map mode;
- `mp_military_presence_assignment_map_value` — mirrored assignment flag used for the striped secondary map color.

Per-army live route state:

- `mp_patrol_active` — scripted patrol assignment state;
- `mp_patrol_route_configured` — scalar GUI-safe flag indicating that a committed route exists;
- `mp_patrol_provinces` — committed ordered membership list;
- `mp_patrol_head` — first province in the route;
- `mp_patrol_tail` — last province in the route;
- `mp_patrol_current` — current virtual patrol province;
- `mp_patrol_next` — variable map implementing an ordered circular linked list (`province -> next province`).

Per-army editor state:

- `mp_patrol_edit_provinces` — temporary ordered working list;
- `mp_patrol_edit_open` — editor working-copy state;
- `mp_patrol_edit_dirty` — whether the working copy was changed;
- `mp_patrol_editor_last_map_focus` — last map target mirrored into the working list.

The monthly driver runs from `monthly_country_pulse`.

The multi-select editor combines:

- a Generic Action with `move_to_next_section_on_click = no`, `top_widget`, `bottom_widget`, `selected`, `map_color`, and `secondary_map_color`;
- a custom province attribute column injected into the target table;
- scripted GUIs for checkbox toggle, map-click mirroring, Area/Region bulk selection, Select All, Apply and Cancel;
- an ordered temporary variable list that is transactionally rebuilt into the live circular route on Apply.

The right-click integration redefines vanilla `UnitContextMenu` and `UnitMarkerContextMenu` in `in_game/gui/aaa_mp_unit_context_menu.gui`. The `aaa_` prefix follows tested EU5 GUI-mod precedent for first-definition-wins loading, and the definitions live in the same `types ContextMenuSpecificTypes` collection as vanilla. Vanilla quick unit actions and the `unit_contextmenu_pre_entries` hook remain preserved.

## Native army objective limitation

EU5's army Actions/Objectives pane renders engine-provided `UnitActionItem` objects. Hardcoded military objectives such as Carpet Siege are not exposed as a scriptable `common/` type that a mod can register a new objective into.

For that reason this MVP does **not** claim to create a new native Carpet-Siege-style engine objective. The patrol controls are Generic Actions integrated into the normal army right-click context menu. Placing a custom button visually inside the same Army Actions/Objectives panel remains a separate compatibility-sensitive GUI patch.

## Known MVP limitations

- No physical army movement or map animation.
- The native hardcoded military-objective registry cannot currently be extended through verified script definitions.
- Only provinces fully owned by the army's country are accepted/processed in this MVP.
- Contributions are fixed per army, not yet scaled by army strength/composition; splitting armies can therefore multiply the contribution in the current MVP.
- No direct instantaneous Control increase is applied; the current implementation modifies monthly Control growth, maximum Control and unrest.
- AI does not configure patrol routes.
- Stationary-army assignment stripes refresh on the monthly cache rebuild rather than on every physical movement frame.
- The `UnitContextMenu` / `UnitMarkerContextMenu` redefinitions are GUI compatibility points with other mods that redefine the same vanilla types.
- The custom selector widgets, map-target watcher and attribute-column injection require in-game validation with `debug_mode` / `error.log` because they are compatibility-sensitive GUI/data hooks.

## Target

Built against the EU5 1.3-era scripting interface. This is an experimental MVP and should be tested with `debug_mode` / script error logging after GUI changes and before balancing or compatibility-sensitive work.
