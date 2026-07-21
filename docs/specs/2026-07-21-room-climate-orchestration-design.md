# Room Climate Orchestration — Design

**Date:** 2026-07-21
**Status:** Approved (pending final spec review)
**Home:** Sioux Falls House (`ha.home.chrisuthe.com`)

## Problem

Four hand-copied automations currently arbitrate five minisplit heads against per-room
hot-water heat. They have drifted apart (inconsistent thresholds, a missing outdoor
upper bound that allows summer heating, two weather sources, mode-only control that
leaves minisplit setpoints frozen at 69–70), and the family has no way to adjust room
temperature: wall thermostats sit `off` all summer and the minisplit setpoints are
pinned by automation. A master toggle (`input_boolean.use_heat_pumps`) gates
everything and must be flipped seasonally by hand.

## Goal

One blueprint, four instances. The wall thermostat in each room becomes the single
family-facing dial, year-round. Home Assistant does the right thing given the dial,
the room temperature, and the outdoor temperature. No seasonal toggles.

## The contract

- Every wall thermostat stays in **heat mode year-round** with a live setpoint (the
  "dial", **D**). The family only ever changes D.
- Room temperature is always read from the **wall thermostat's sensor** — never the
  minisplit head sensor, which over-reads in heating (warm air stratifies at the
  ceiling). This also matters because the wall sensor is what triggers the hardwired
  boiler call.
- Hot-water heat is hardwired (thermostat relay → zone valve → boiler); HA cannot
  intercept a call. The boiler is physically off in summer (manual, outside HA's
  view). Arbitration is therefore *emergent*: the heat pump holds the room just above
  the wall trigger so the boiler is never called; if the heat pump can't keep up, the
  room sags to the dial and the hardwired path takes over as backup.

## Zones

| Zone | Dial (wall thermostat) | Minisplit head(s) |
|---|---|---|
| Main Floor | `climate.living_room_livingroomthermostat_climate` | `climate.main_floor_east`, `climate.main_floor_west` |
| Master Bedroom | `climate.master_bedroom_thermostat_fan_thermostat` | `climate.master_bedroom_climate` |
| Margaret's Room | `climate.zen_within_zen_01_1d6f1200_fan_thermostat` | `climate.air_conditioner_margaret_margaret_air_conditioner` |
| Will's Room | `climate.zen_within_zen_01_9cca1200_fan_thermostat` | `climate.air_coniditioning_will_wills_bedroom` |

The Family Room keeps its Zen thermostat and hot-water heat only; it has no minisplit
and is out of scope. Garage and IR-blaster climate entities are out of scope.

## Mode logic

With D = dial setpoint, boost = the room's boost helper value (default 0), and
cal = per-room calibration offset:

| State | Entry condition | Minisplit action |
|---|---|---|
| **Cool** | room ≥ D + `cool_band` (1.0°) AND outdoor ≥ `cool_outdoor_floor` (60°) | `cool`, target D − 0.5, clamped to 62.5–86 |
| **HP heat** | room ≤ D + 0.5 + boost AND `hp_outdoor_floor` (20°) ≤ outdoor < `hp_outdoor_max` (60°) | `heat`, target D + `hp_offset` (1.0) + boost + cal, clamped |
| **Idle** | otherwise | `idle_mode` input: `off` or `fan_only` |

Anti-fight guarantees:

- The outdoor windows for HP heat and cooling are **disjoint at 60°** — only one side
  is ever armed, so the tight indoor bands (cool entry at D+1 vs heat target D+1)
  cannot seesaw.
- Room-temperature triggers require ~5 minutes of persistence; outdoor-crossing
  triggers ~15 minutes.
- Cool target D − 0.5 stays above the Zen call-for-heat differential (~1° below D),
  so an AC cycle does not poke the boiler in shoulder season. If observation shows a
  Zen with a tighter differential, raise the cool target for that room.

Notes on semantics:

- "Dial = desired temperature" holds year-round: a cool night below 60° outdoors with
  the room under the dial gets heat-pump heat. That is intended; the family remedy is
  turning the dial down.
- `input_boolean.use_heat_pumps` is retired. The outdoor window *is* the gate.

## Blueprint inputs

| Input | Type | Default |
|---|---|---|
| `dial_thermostat` | climate entity | — |
| `minisplits` | climate entity, multiple | — |
| `weather_entity` | weather entity | `weather.pirateweather` (all rooms standardized) |
| `cool_band` | number (°F) | 1.0 |
| `cool_outdoor_floor` | number (°F) | 60 |
| `hp_offset` | number (°F) | 1.0 |
| `hp_outdoor_floor` | number (°F) | 20 |
| `hp_outdoor_max` | number (°F) | 60 |
| `calibration_offset` | number (°F) | 0 (per-room, measured head-vs-wall disagreement) |
| `idle_mode` | select: `off` / `fan_only` | `off` |
| `boost_helper` | input_number entity (optional) | none |
| `lockout_switch` | switch entity (optional, Phase 2) | none |

Triggers: dial setpoint change, dial `current_temperature` change (with persistence),
outdoor temperature change (with persistence), minisplit availability change,
boost helper change, HA start / automation reload. Mode: `restart`.

## Morning warm-ups (rebuilt)

Replaces `automation.warm_up_master_bedroom` and `automation.warm_up_the_kids`
(delay-based; not restart-safe; moved wall setpoints, which under the new contract
would fire the boiler).

- Per-room `input_number` boost helpers: `input_number.climate_boost_master`,
  `input_number.climate_boost_margaret`, `input_number.climate_boost_will`.
- One schedule automation with four independent time triggers (restart-proof):
  - 05:30 daily → master boost = 3; 06:30 → master boost = 0
  - 06:30 weekdays → both kids' boosts = 2; 07:30 → both = 0
- Boost values preserve the old warm-up targets: master 66+1+3 = 70, kids 67+1+2 = 70.
- Boost raises both the HP-heat entry threshold and target, so the minisplit does the
  morning bump alone; the wall dial (and thus the boiler trigger) never moves.
- Schedule is gated on outdoor temperature < 45° so summer mornings are unaffected.

## Phase 2 — boiler lockout interlock (hardware ordered, not yet arrived)

Seeed Studio XIAO 6-Channel Wi-Fi Relay (ESP32-C6, six relays with COM/NO/NC,
contacts 10A @ 250VAC, ships with ESPHome). Each zone's thermostat call-for-heat pair
is broken through a relay's **COM+NC** contacts at the zone panel: de-energized =
circuit closed = heat allowed. Energize to block — every failure mode (power, Wi-Fi,
ESP crash, HA down) decays to "boiler available".

Custom ESPHome firmware requirements:

- `restore_mode: ALWAYS_OFF` on all relay outputs (reboot never blocks heat)
- On HA API disconnect → de-energize all relays
- Local auto-release: a lockout not renewed by HA within 30 minutes releases itself

Blueprint behavior when `lockout_switch` is set: engage lockout on entering HP heat;
release on exiting HP heat, on room < D − 1° sustained 20 minutes (HP can't keep up —
let the boiler in), on outdoor < `hp_outdoor_floor`, on minisplit unavailable, or on
room < 60° (freeze guard). Until the hardware is installed the input stays empty and
recovery events simply get brief boiler assist (both sources run until the room
crosses the wall trigger; ~20–40 min, arguably good UX).

## Deployment

- Public repo `chrisuthe/ha-blueprints`: blueprint YAML, this spec, later the
  ESPHome config. Imported into HA via raw-URL blueprint import; updates re-imported.
- Helpers, blueprint-instance automations, and the schedule automation are created
  via the REST config API (long-lived token) — no config file access needed.

## Rollout

1. Create repo, author blueprint, import into HA.
2. One-time prep: set all four wall thermostats to heat mode with initial dials equal
   to the **current minisplit targets** (main floor 69, master 69, kids 70) so day-one
   behavior matches today; the family tunes from there. (The winter hot-water numbers
   66–67 would drive the AC to ~65.5° in July.)
3. Create helpers + schedule automation.
4. **Pilot: Will's room.** Disable (don't delete) `automation.minisplit_control_wills_room`.
   Cooling path is live-testable in July; heating path exercised synthetically by
   raising the dial. Observe 1–2 days.
5. Roll remaining three rooms; disable their old automations.
6. After validation: delete the four old automations, both old warm-ups, and
   `input_boolean.use_heat_pumps`.
7. Phase 2 when the relay board arrives: flash + bench-test watchdog behavior, wire
   at the zone panel, fill `lockout_switch` on each instance.

## Open items

- Measure each head's calibration offset (head reading vs wall reading) after a few
  days of running and set per-room `calibration_offset`.
- Observe the Zens' actual call-for-heat differential once the boiler returns in fall.
- Optional later: a "keeper" automation restoring heat mode if a wall stat gets
  turned off out of habit.
