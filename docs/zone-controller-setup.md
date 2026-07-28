# Zone Controller — Home Assistant Setup Guide

This guide sets up per-room zoning for a ducted HVAC system using two Home
Assistant blueprints plus a set of helper entities. The firmware (ESPHome /
KC868) is separate — this is purely the Home Assistant control logic. For the
big-picture overview and architecture diagram, see the [README](../README.md).

## How it works

One upstairs unit serves four rooms through motorized dampers:

| Priority | Room | Damper(s) | Mode | Setpoints |
|:--:|------|:--:|------|-----------|
| 1 | Nursery | 1 | standard, tight band | Heat < 68 °F, Cool > 71 °F |
| 2 | Master bedroom | 2 | occupancy, cool-only | Cool > 75 °F day / > 68 °F Sleep |
| 3 | Server room | 1 | always-on, cool-only | Cool > 80 °F |
| 4 | Theater / bonus | 1 | relief valve | Coordinator-managed |
| — | Hallway | 0 | always open (structural relief) | — |

Each conditioned room runs a **Room Zone** automation. It computes a raw demand
(`heat` / `cool` / `none`), writes it to an `input_text` helper, and opens its
damper only when its demand matches the direction the unit is delivering. The
single **Coordinator** automation reads those demand helpers *in priority order*,
sets the unit's direction to the highest-priority calling room, turns the unit
on/off, drives the theater damper as a relief valve, and handles home/away.

Because a single air handler is hot **or** cold at any moment, priority decides
which room "wins" the unit when demands conflict; lower-priority rooms wanting the
opposite direction wait with their dampers closed.

> **Damper polarity is inverted / normally-open:** `off` = damper **OPEN**,
> `on` = damper **CLOSED**. The blueprints account for this; a power loss springs
> every damper open (failsafe). You only point them at the correct `switch`
> entities.

## Prerequisites

You need these entities to already exist in Home Assistant:

- **Room temperature sensors** — one per room (`sensor.*`, device_class `temperature`).
- **Master bedroom occupancy** — a `binary_sensor.*` that is `on` when occupied.
- **Damper switches** — one `switch.*` per damper (5 total; master bedroom has 2).
- **Main HVAC** — a `climate.*` entity that supports `cool`, `heat`, `heat_cool`, `off`.

## 1. Install the helper package

Copy [`packages/zone_controller.yaml`](../packages/zone_controller.yaml) into
your HA config at `<config>/packages/zone_controller.yaml`. Enable packages in
`configuration.yaml` if you haven't already:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Edit the package and replace every `# TODO` (the `zone:` latitude/longitude, and
confirm the setpoint defaults). Then **restart Home Assistant**. This creates:

- Setpoints: `input_number.nursery_heat_sp` / `nursery_cool_sp`,
  `server_cool_sp`, `master_cool_sp` / `master_sleep_cool_sp`
- Enable toggles: `input_boolean.nursery_enabled`, `server_enabled`, `master_enabled`
- Demand helpers: `input_text.nursery_demand`, `master_demand`, `server_demand`
- `input_datetime.zc_last_changeover`
- Setpoint force-run: `input_boolean.zc_setpoint_manual`,
  `input_number.zc_last_cmd_setpoint`, `input_datetime.zc_manual_until`
- `zone.home_wide` (~5 mi)

> The **sleep-mode** and **home-occupied** booleans are **not** created by the
> package — use your existing helpers. The steps below refer to them as
> `input_boolean.<your_sleep_mode>` and `input_boolean.<your_home_occupied>`;
> substitute your own entity IDs.

## 2. Import the blueprints

**Settings → Automations & Scenes → Blueprints → Import Blueprint**, then import
both (or copy into `<config>/blueprints/automation/zone_controller/`):

- `blueprints/automation/zone_controller/room_zone.yaml`
- `blueprints/automation/zone_controller/zone_coordinator.yaml`

## 3. Create the room automations

Create **one automation per conditioned room** from the **Room Zone** blueprint.

### Nursery (priority 1 — standard, tight band)
- Room enable toggle: `input_boolean.nursery_enabled`
- Demand helper: `input_text.nursery_demand`
- Room temperature sensor: `sensor.nursery_temperature`
- Damper switch(es): `switch.damper_nursery`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.nursery_cool_sp` · Enable cooling: **on**
- Heat setpoint: `input_number.nursery_heat_sp` · Enable heating: **on**
- Hysteresis: **0.3** (half-band on each side → the room settles within ~0.6° of
  the setpoint) · Zone mode: **Standard**

### Master bedroom (priority 2 — occupancy, cool-only, sleep, 2 dampers)
- Room enable toggle: `input_boolean.master_enabled`
- Demand helper: `input_text.master_demand`
- Room temperature sensor: `sensor.master_bedroom_temperature`
- Damper switch(es): `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.master_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- Zone mode: **Occupancy** · Occupancy sensor: `binary_sensor.master_bedroom_occupancy`
- Sleep mode boolean: `input_boolean.<your_sleep_mode>`
- Sleep / pre-cool cool setpoint: `input_number.master_sleep_cool_sp`
- **Scheduled activation** (pre-cool for sleep):
  - Active after: **20:00:00** · Active until: **07:00:00**
  - Schedule requires house occupied: `input_boolean.<your_home_occupied>`

  This makes the master bedroom start pre-cooling around 8pm whenever the house
  is occupied, on top of its occupancy behaviour (the window runs overnight to
  07:00). Inside the window the room automatically switches to the **68 °F**
  sleep/pre-cool setpoint — you do **not** need to flip Sleep mode; the toggle is
  just a manual override to pull the target down at any other time. So at, say,
  9pm and 74° the room will already be cooling toward 68. It becomes active if
  the room is occupied **or** it's after 8pm and someone is home. Latency is up
  to the ~5 min room resync, hence "around" 8pm.

### Server room (priority 3 — always on, cool-only)
- Room enable toggle: `input_boolean.server_enabled`
- Demand helper: `input_text.server_demand`
- Room temperature sensor: `sensor.server_room_temperature`
- Damper switch(es): `switch.damper_server_room`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.server_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- Zone mode: **Always on**

> The theater / bonus room does **not** get a Room Zone automation — its damper
> is managed only by the coordinator.

## 4. Create the coordinator automation

Create **one** automation from the **Coordinator** blueprint:

- Main HVAC climate entity: `climate.upstairs_hvac`
- **Room demand helpers, HIGHEST PRIORITY FIRST** — add in this order:
  `input_text.nursery_demand`, `input_text.master_demand`,
  `input_text.server_demand`. **Order = priority.**
- Room damper switches (all non-relief): `switch.damper_nursery`,
  `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`,
  `switch.damper_server_room`
- Relief valve damper switch: `switch.damper_theater`
- Compressor reversal cooldown: **3** (minutes)
- Last-changeover helper: `input_datetime.zc_last_changeover`
- Home occupied toggle: `input_boolean.<your_home_occupied>`
- Wide presence zone: `zone.home_wide`
- Away action: **Set eco preset** *(keeps the unit running so the server room
  stays cooled while the house is empty)*
- Away preset name / Home preset name: match your unit's presets (defaults
  `eco` / `none`).
- **Setpoint Force-Run** (optional but recommended — see below):
  - Manage the unit's setpoint: **on**
  - Force-run offset: **2** · Setpoint floor: **60** · Setpoint ceiling: **85**
  - Manual setpoint override toggle: `input_boolean.zc_setpoint_manual`
  - Manual-edit grace: **30** (minutes)
  - Last-commanded setpoint helper: `input_number.zc_last_cmd_setpoint`
  - Manual-grace-until helper: `input_datetime.zc_manual_until`

> Choose **Turn the unit off** for the away action only if you have no room that
> must be conditioned regardless of occupancy — it stops the server room too.

### Why Setpoint Force-Run?

The central thermostat senses the **hallway**. If the hallway is 67 °F, a room
wants cooling at 71 °F, and the thermostat's setpoint is still 72 °F, the unit
**won't start** — the hallway is already "cool enough" as far as the thermostat
is concerned. With force-run enabled, whenever a room wins the arbitration the
coordinator pushes the unit's target a couple of degrees past the sensed hallway
temperature (e.g. cool → 65 °F) so it actually runs; the room dampers then do the
real regulation and the coordinator turns the unit off once every room is
satisfied. The target is clamped between the floor and ceiling.

**Keeping manual control.** Two ways to override:
- **Hard:** turn `input_boolean.zc_setpoint_manual` **on** — the coordinator
  never touches the setpoint until you turn it back off.
- **Soft:** just change the setpoint by hand (toggle off). The coordinator
  detects the manual edit and leaves it alone for the grace period (default 30
  min) before resuming force-run. So a manual tweak is never overridden
  immediately.

## 5. Verify

1. **Demand + damper.** Set `input_number.nursery_cool_sp` below the nursery
   temperature. `input_text.nursery_demand` should become `cool`; with the unit
   cooling, `switch.damper_nursery` goes **off** (open). Raise the setpoint back
   and demand returns to `none`, damper **on** (closed).
2. **Enable/disable.** Turn off `input_boolean.nursery_enabled`:
   `input_text.nursery_demand` should go `none` and `switch.damper_nursery`
   **on** (closed), and the nursery drops out of arbitration.
3. **Priority conflict.** Make the nursery call for heat while the master calls
   for cooling (adjust setpoints). The unit should go/stay in **heat** for the
   nursery; the master's dampers stay **on** (closed) until the nursery is
   satisfied. Priority is never yielded to the lower-priority master — the only
   thing that can briefly delay the switch to heat is the compressor cooldown.
4. **Compressor reversal cooldown.** Watch `input_datetime.zc_last_changeover` —
   after a direction change, the unit won't reverse heat↔cool again until
   `min_changeover` minutes pass (unless no room is still calling the current
   direction). With the default of 3 min, the nursery is served ~3 min after the
   last reversal at worst; set the cooldown to 0 for instant reversal.
5. **Relief valve.** Force every room damper closed (each switch `on`):
   `switch.damper_theater` goes **off** (open). Open any room damper and the
   theater damper goes **on** (closed).
6. **Main on/off.** With a room calling, the unit switches on in the winning
   direction. With every room satisfied/disabled, it switches off.
7. **Sleep / pre-cool.** Turn on `input_boolean.<your_sleep_mode>`; the master
   bedroom uses the 68 °F sleep cool setpoint. The same 68 °F target applies
   automatically inside the 20:00–07:00 window (house occupied) without touching
   the toggle — so an evening reading above 68 begins pre-cooling on its own.
8. **Away.** Turn off `input_boolean.<your_home_occupied>` with nobody in
   `zone.home_wide`; the configured away action fires (eco preset by default).
9. **Force-run.** With force-run enabled, set the central thermostat to a mild
   value (e.g. 72) while a room calls for cooling and the hallway is cooler than
   that. The coordinator should drive the target down (≈ sensed − 2) so the unit
   runs, and `input_number.zc_last_cmd_setpoint` updates to that value.
10. **Manual respect.** Change the setpoint by hand: it should stick for the
    grace period (`input_datetime.zc_manual_until` set into the future) rather
    than being corrected on the next cycle. Turning
    `input_boolean.zc_setpoint_manual` on holds it indefinitely.

### Template sanity check

In **Developer Tools → Template**, paste (substituting your entity IDs) to
confirm the coordinator's core expressions:

```jinja
{% set zone_demands = ['input_text.nursery_demand','input_text.master_demand','input_text.server_demand'] %}
{% set zone_dampers = ['switch.damper_nursery','switch.damper_master_bedroom_1','switch.damper_master_bedroom_2','switch.damper_server_room'] %}
winner_dir:        {% set ns = namespace(dir='none') %}{% for d in zone_demands %}{% if ns.dir=='none' and states(d) in ['heat','cool'] %}{% set ns.dir=states(d) %}{% endif %}{% endfor %}{{ ns.dir }}
all_others_closed: {{ zone_dampers | reject('is_state','off') | list | count == (zone_dampers | count) }}
```

## Notes & limitations

- **Rooms publish demand; the Coordinator owns direction.** Because arbitration
  requires it, the coordinator fully controls the unit's `hvac_mode` — a manual
  mode change will be overridden.
- Keep each room's **heat setpoint at least ~2° below its cool setpoint** so a
  room can't be cooled into needing heat (or vice-versa) and thrash.
- A room whose demand is the **opposite** of the current unit direction waits with
  its damper closed until it wins arbitration or the direction is free.
- Optional inputs (occupancy, sleep, presence zone) are re-evaluated on the
  periodic resync (coordinator every 2 min, rooms every 5 min), so worst-case
  latency for those is a couple of minutes — fine for HVAC. Enable toggles and
  demand changes are instant (they are triggers).
