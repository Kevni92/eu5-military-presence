# EU5 Military Presence

Experimental Europa Universalis V mod that adds a scripted **Military Presence** system for land provinces and a virtual army patrol assignment.

## MVP scope

This first implementation deliberately models patrols **virtually**. Armies do not receive physical movement orders and their map position is not changed.

A patrol army maintains an ordered, circular route of provinces. While the `Military Patrol` unit ability is active, the army contributes Military Presence to its current virtual province. Once that province reaches the configured threshold (90), the army advances to the next province in its route. After the last province it wraps back to the first.

### Route setup

The MVP uses vanilla Generic Actions instead of a custom multi-select GUI:

1. Use **Add Military Patrol Province**.
2. Select one of your armies.
3. Select a fully owned province.
4. Repeat to append as many provinces as desired.
5. Toggle **Military Patrol** on for that army.

The insertion order is the patrol order. The route length itself is not fixed; the repeated action can append an arbitrary number of provinces. **Clear Military Patrol Route** removes the complete route from a selected army.

## Current balance

- Military Presence range: `0..100`
- Patrol switch threshold: `90`
- Active patrol gain: `+12` per month
- Passive decay: `-2` per month
- At 100 Military Presence, the province modifier provides:
  - `-0.10` local unrest
  - `+0.01` local monthly control

The modifier scales linearly with Military Presence.

## Technical design

Per-province state:

- `mp_military_presence` — numeric Military Presence value.

Per-army state:

- `mp_patrol_active` — set while the unit ability is active.
- `mp_patrol_provinces` — membership list used for duplicate prevention.
- `mp_patrol_head` — first province in the route.
- `mp_patrol_tail` — last province in the route.
- `mp_patrol_current` — current virtual patrol province.
- `mp_patrol_next` — variable map implementing an ordered circular linked list (`province -> next province`).

The monthly driver runs from `monthly_country_pulse`.

## Known MVP limitations

- No physical army movement or map animation.
- Province selection is one province per Generic Action invocation; there is no custom multi-select UI yet.
- Only provinces fully owned by the army's country are accepted/processed in this MVP.
- Patrol contribution is a fixed monthly amount, not yet scaled by army strength/composition.
- No direct instantaneous Control increase is applied; the current implementation modifies monthly Control growth and unrest.
- AI does not configure patrol routes.
- Route ownership changes are handled conservatively by skipping/advancing from an invalid current province; route editing is otherwise manual.

## Target

Built against the EU5 1.3-era scripting interface. This is an experimental MVP and should be tested with `debug_mode` / script error logging before balancing or UI work.
