# EU5 Military Presence

Experimental Europa Universalis V mod that adds a scripted **Military Presence** system for land provinces and a virtual army patrol assignment.

## MVP scope

Every non-exiled army now generates Military Presence automatically in the fully owned province where it is physically stationed. No patrol route or player action is required for this baseline effect.

Patrols are modeled **virtually**. Armies do not receive physical movement orders and their map position is not changed.

A patrol army maintains an ordered, circular route of provinces. While the army is assigned to Military Patrol duty, it contributes Military Presence to its current virtual province. Once that province reaches the configured threshold (90), the army advances to the next province in its route. After the last province it wraps back to the first.

Military Patrol is not implemented as a `unit_ability`. The assignment is stored as scripted state on the army (`mp_patrol_active`). An army with an active patrol contributes through its virtual patrol route instead of simultaneously contributing at its physical location, preventing double counting of the same army.

### Using the system in game

No configuration is required for normal stationed armies:

1. Station one of your armies in a province that you fully own.
2. Let a monthly pulse pass.
3. Open the **Military Presence** map mode in the Military map-mode category.
4. The province should gain Military Presence each month while the army remains there.

To configure a virtual patrol route:

1. Use **Add Military Patrol Province**.
2. Select one of your armies.
3. Select a fully owned province.
4. Repeat to append as many provinces as desired.
5. Use **Start Military Patrol** for that army.

Use **Stop Military Patrol** to suspend the assignment without deleting the route. **Clear Military Patrol Route** removes the complete route.

The insertion order is the patrol order and route length is not fixed.

## Military Presence map mode

A custom **Military Presence** map mode is defined under `in_game/gfx/map/map_modes/` in the Military category.

Military Presence remains authoritative at province scope. Because EU5 map-mode coloring is evaluated per location, the current province value is mirrored to all locations in that province as `mp_military_presence_map_value`. This makes the whole province display the same color.

- no presence: grey
- low presence: red
- high presence: green
- tooltip: current value from 0 to 100

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

The province modifier scales linearly with Military Presence. Therefore 50 Presence gives half of those effects, including +5% maximum Control; 100 Presence gives +10% maximum Control.

## Technical design

Per-province state:

- `mp_military_presence` — authoritative numeric Military Presence value.

Per-location display cache:

- `mp_military_presence_map_value` — mirrored value used only by the map mode.

Per-army state:

- no patrol state is required for normal physical presence;
- `mp_patrol_active` — scripted patrol assignment state;
- `mp_patrol_provinces` — membership list used for duplicate prevention;
- `mp_patrol_head` — first province in the route;
- `mp_patrol_tail` — last province in the route;
- `mp_patrol_current` — current virtual patrol province;
- `mp_patrol_next` — variable map implementing an ordered circular linked list (`province -> next province`).

The monthly driver runs from `monthly_country_pulse`.

## Native army objective limitation

EU5's army Actions/Objectives pane renders engine-provided `UnitActionItem` objects. Hardcoded military objectives such as Carpet Siege are not exposed as a scriptable `common/` type that a mod can register a new objective into.

For that reason this MVP does **not** claim to create a new native Carpet-Siege-style engine objective. Start/Stop Military Patrol are scripted actions. Placing a custom button visually inside the same Army Actions/Objectives panel requires overriding `single_unit_window.gui`, which should be treated as a separate compatibility-sensitive GUI patch.

## Known MVP limitations

- No physical army movement or map animation.
- The native hardcoded military-objective registry cannot currently be extended through verified script definitions.
- Province selection is one province per Generic Action invocation; there is no custom multi-select UI yet.
- Only provinces fully owned by the army's country are accepted/processed in this MVP.
- Contributions are fixed per army, not yet scaled by army strength/composition; splitting armies can therefore multiply the contribution in the current MVP.
- No direct instantaneous Control increase is applied; the current implementation modifies monthly Control growth, maximum Control and unrest.
- AI does not configure patrol routes.

## Target

Built against the EU5 1.3-era scripting interface. This is an experimental MVP and should be tested with `debug_mode` / script error logging before balancing or compatibility-sensitive GUI work.
