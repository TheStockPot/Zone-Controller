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
| 4 | Theater / bonus | 1 | guest room (when guest mode on) + relief valve | Cool > 74 °F day / > 69 °F sleep · pre-cools from 20:00 |
| — | Hallway | 0 | always open (structural relief) | — |

Each room — the theater included — runs a **Room Zone** automation. It computes a
raw demand (`heat` / `cool` / `none`), writes it to an `input_text` helper, and
opens its damper only when its demand matches the direction the unit is
delivering. The single **Coordinator** automation reads those demand helpers *in
priority order*, sets the unit's direction to the highest-priority calling room,
turns the unit on/off, decides when the theater must open as the pressure-relief
valve (publishing a flag the theater's Room Zone obeys), and handles home/away.

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
  `server_cool_sp`, `master_cool_sp` / `master_sleep_cool_sp` /
  `master_away_cool_sp`, `theater_cool_sp` / `theater_sleep_cool_sp`
- Enable toggles: `input_boolean.nursery_enabled`, `server_enabled`,
  `master_enabled`, `theater_guest_mode` (the theater's enable)
- Comfort boosts: `input_boolean.master_boost`, `theater_boost`
- Demand helpers: `input_text.nursery_demand`, `master_demand`, `server_demand`,
  `theater_demand`; urgent helper: `nursery_urgent`; per-room status:
  `nursery_status`, `master_status`, `server_status`, `theater_status`
- `input_datetime.zc_last_changeover`
- Setpoint force-run: `input_boolean.zc_setpoint_manual`,
  `input_number.zc_last_cmd_setpoint`, `input_datetime.zc_manual_until`
- Full manual override: `input_boolean.zc_bypass`
- Cooling relief flag: `input_boolean.zc_cool_relief`
- Theater relief-valve flag: `input_boolean.zc_theater_relief`
- Away circulation purge: `input_datetime.zc_purge_until`
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

Create **one automation per room** from the **Room Zone** blueprint — four in
total: nursery, master bedroom, server room, and the theater.

Each room also has an optional **Room status output** (`input_text`) it writes a
plain-English explanation to — its temperature vs its effective target, day/sleep,
whether it's yielding or disabled — so you can see *why* each room is or isn't
calling. Point each room at its own (`nursery_status`, `master_status`,
`server_status`, `theater_status`).

Also set every room's **Full manual override / bypass** input to the shared
`input_boolean.zc_bypass` (the same one the Coordinator uses). While that toggle
is on, the room stops driving its damper so you have full manual control; off, it
resumes. (Omitted from the per-room lists below for brevity — set it on each.)

### Nursery (priority 1 — standard, tight band)
- Room enable toggle: `input_boolean.nursery_enabled`
- Demand helper: `input_text.nursery_demand`
- Room status output: `input_text.nursery_status`
- Room temperature sensor: `sensor.nursery_temperature`
- Damper switch(es): `switch.damper_nursery`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.nursery_cool_sp` · Enable cooling: **on**
- Heat setpoint: `input_number.nursery_heat_sp` · Enable heating: **on**
- Hysteresis: **0.3** (half-band on each side → the room settles within ~0.6° of
  the setpoint) · Zone mode: **Standard**
- **Priority Yield** section → *Urgent-demand output*: `input_text.nursery_urgent`
  · *Urgent margin*: **2** — the nursery publishes `cool`/`heat` here only while
  it's more than 2° past its setpoint, so lower-priority rooms yield to it **only
  in extreme cases** (e.g. 75° when it needs 70°) and stop once it recovers to
  ~72°. (Leave its own "Yield to..." unset — the nursery never yields.)

### Master bedroom (priority 2 — occupancy, cool-only, sleep, 2 dampers)
- Room enable toggle: `input_boolean.master_enabled`
- Demand helper: `input_text.master_demand`
- Room status output: `input_text.master_status`
- Room temperature sensor: `sensor.master_bedroom_temperature`
- Damper switch(es): `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.master_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- **Priority Yield** *(optional)*: *"Yield to this higher-priority room's demand
  helper"* → `input_text.nursery_urgent`. The master then closes its damper only
  while the nursery is **urgent** (far past its target), concentrating air on the
  nursery until it recovers to ~72°, then reopens — while both keep calling. (Point
  it at `input_text.nursery_demand` instead if you want the master to yield
  *whenever* the nursery is served, not just in extreme cases; leave unset for no
  yielding at all.)
  - **Only yield during the day (not at night):** **on**. During the day the
    master yields to the nursery as above; **at night** (on its sleep/pre-cool
    setpoint) the yield is suspended, so the master keeps its damper **open** and
    serves as the cooling-relief zone while it cools toward its own sleep target —
    then closes once it's satisfied, handing relief to the server.
- Zone mode: **Occupancy** · Occupancy sensor: `binary_sensor.master_bedroom_occupancy`
- Sleep mode boolean: `input_boolean.<your_sleep_mode>`
- Sleep / pre-cool cool setpoint: `input_number.master_sleep_cool_sp`
- **Comfort boost toggle** *(optional)*: `input_boolean.master_boost` — flip it on
  to temporarily cool the master to its **sleep setpoint (68°)** during the day
  (e.g. someone's getting ready in there). While on, the room uses the colder
  target **and** stays eligible even though it's occupancy-gated, so it cools
  regardless of the clock or whether the occupancy sensor sees anyone. Turn it off
  to return to the normal day target. Its status shows `(boost)`.
- **Away Setpoint Override:** Home-occupied toggle → `input_boolean.<your_home_occupied>`
  · Away cool setpoint → `input_number.master_away_cool_sp` (**75**). While away
  (home toggle off) the master stays eligible **even though it's occupancy-gated**
  and is capped at 75° — so it can't bake to 78 while you're out, but still runs
  warmer than its day target to save power.
- **Scheduled activation** (pre-cool for sleep):
  - Active after: **20:00:00** · Active until: **07:00:00**
  - Schedule requires house occupied: `input_boolean.<your_home_occupied>`
  - Start the window earlier when this is on: `input_boolean.theater_guest_mode`
    · Early-start minutes: **60** — when the theater guest room is running (extra
    load, less airflow per room), pre-cool begins at **19:00** instead of 20:00 so
    the bedroom still reaches its sleep target on time.

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
- Room status output: `input_text.server_status`
- Room temperature sensor: `sensor.server_room_temperature`
- Damper switch(es): `switch.damper_server_room`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.server_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- Zone mode: **Always on**
- **Cooling Ride-Along:** *"Ride along while cooling"* = **on** · *Ride-along floor
  below cool setpoint* = **4** · **_"Ride along only while this is on"_** =
  `input_boolean.zc_cool_relief`. This makes the server the preferred second
  airflow path — **but only when a second path is actually needed.** While the
  nursery and master are *both* calling (2 zones already open), the server stays
  **closed** so the full airflow goes to the bedrooms. The moment only one room is
  left calling, the coordinator turns `zc_cool_relief` on and the server rides
  along (down to ~4° below its setpoint) as the second zone — cooling relief the
  server benefits from, instead of wasting it on the theater. The theater only
  opens as a last resort (e.g. the server is already below its floor).
  > This makes the **server the default cooling-relief path** — whenever fewer
  > than 2 *primary* zones are actually open, the server opens (never the theater).
  > That includes the daytime case where the master has **yielded** to the nursery
  > (only the nursery is open) — the server relieves, not the theater. **At night**
  > the master doesn't yield, so the master is the relief zone until it reaches its
  > sleep target and closes, and then the server takes over — all via this one
  > flag, no extra server wiring. Leave the gate unset to ride along on every
  > cooling cycle (the older behavior).
- **Priority Yield:** *leave unset.* The server must never stop getting air, so it
  should not yield to the nursery (or any room).

### Theater / bonus (priority 4 — guest room + relief valve)
The theater has two jobs: a **guest bedroom** while guest mode is on, and the
**last-resort pressure-relief valve** at all times. Its guest-mode toggle is its
Enable toggle, so with guest mode off it publishes no demand and only opens when
the coordinator says relief is needed.
- Room enable toggle: `input_boolean.theater_guest_mode`
- Demand helper: `input_text.theater_demand`
- Room status output: `input_text.theater_status`
- Room temperature sensor: `sensor.theater_temperature` *(your theater sensor)*
- Damper switch(es): `switch.damper_theater`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.theater_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset for cool-only, like the master)* · Enable heating: **off**
- Zone mode: **Standard** — while guest mode is on, the room is always eligible.
- Sleep mode boolean: `input_boolean.<your_sleep_mode>` *(optional — same toggle as the master)*
- Sleep / pre-cool cool setpoint: `input_number.theater_sleep_cool_sp`
- **Comfort boost toggle:** `input_boolean.theater_boost` — temporarily drops the
  theater to its sleep setpoint, like the master's boost. Only works while guest
  mode is on (with guest mode off the theater isn't a bedroom, so boost is ignored).
- **Scheduled activation** (pre-cool, same as the master):
  - Active after: **20:00:00** · Active until: **07:00:00**
  - Schedule requires house occupied: `input_boolean.<your_home_occupied>`
  - Leave *"Start the window earlier…"* unset (that's for the master).

  With guest mode on, from 20:00 the theater switches to its sleep setpoint and
  pre-cools for the guest, exactly like the master. With guest mode off, the
  schedule does nothing (the room is disabled).
- **Priority Yield** — keep the guest room **last**:
  - *Yield to this higher-priority room's demand helper:* `input_text.nursery_demand`
  - *Also yield to these demand helpers:* `input_text.master_demand`
  - The theater closes its damper while the nursery **or** master is being served,
    and only gets air once neither needs it. It still publishes its demand, so the
    unit keeps running for it.
- **Away Circulation Purge → Circulation purge window:** `input_datetime.zc_purge_until`
- **Away Circulation Purge → Force open while this is on:**
  `input_boolean.zc_theater_relief` — **required.** This is how the theater stays
  the relief valve: the coordinator turns this flag on whenever the duct needs a
  relief path, and the theater opens regardless of guest mode or yielding.

> **Trade-off to know:** because the guest room yields to the master, on a night
> when the master is pre-cooling to its sleep target for a long time the theater
> waits its turn and may reach its own target later. The master's *early start*
> (19:00 when guest mode is on) helps the master finish sooner. If the guest's
> comfort matters more than strict ordering, drop `input_text.master_demand` from
> the theater's yield list.

> **If you use the Away Circulation Purge** (below), also set on **every** Room
> Zone: *Away Circulation Purge → Circulation purge window* =
> `input_datetime.zc_purge_until` (the same helper the coordinator uses). That's
> what lets each room open its damper during a purge so air moves through its
> duct. Leave it unset on any room you want to exclude from purges.

## 4. Create the coordinator automation

Create **one** automation from the **Coordinator** blueprint:

- Main HVAC climate entity: `climate.upstairs_hvac`
- **Room demand helpers, HIGHEST PRIORITY FIRST** — add in this order:
  `input_text.nursery_demand`, `input_text.master_demand`,
  `input_text.server_demand`, `input_text.theater_demand` **last**. **Order =
  priority** — the theater goes last so it's the lowest-priority zone. (It only
  publishes demand while guest mode is on, so leave it listed permanently.)
- Room damper switches (all non-relief): `switch.damper_nursery`,
  `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`,
  `switch.damper_server_room` — **not** the theater damper (its relief opening
  would then count toward the minimum and make it flip open/closed).
- **Theater relief output:** `input_boolean.zc_theater_relief` — the coordinator
  turns this on whenever the theater must open as the relief valve. The theater's
  Room Zone opens its damper on it (its *Force open while this is on*).
- Relief valve damper switch *(legacy)*: **leave empty.** Only for the old setup
  where the coordinator drove the theater damper itself — setting it now would
  fight the theater's own Room Zone.
- Minimum open zones while cooling: **2** — while cooling, keep at least 2 zones
  open so the unit never runs through a single small zone (high static pressure).
  The **server's Cooling Ride-Along** normally provides the second zone (useful
  cooling); this theater relief only opens as a **backstop** when there still
  aren't 2 zones open (e.g. the server is already cold). Set to **1** for the
  original behavior. Heating is unaffected.
- Cooling-relief-needed output *(optional but recommended)*:
  `input_boolean.zc_cool_relief` — the coordinator turns this **on** while cooling
  with fewer than the minimum **primary** zones open. Point the server's *"Ride
  along only while this is on"* at it (above) so the server becomes the **default
  relief path**: it opens whenever a second zone is needed — including when the
  master has yielded to the nursery — so relief goes to the server, never the
  theater.
- Relief (ride-along) room damper(s) to exclude: `switch.damper_server_room` — the
  server's own damper, so its relief opening isn't counted as satisfying the
  minimum (otherwise it would flip open/closed). Set this to the same switch as
  the server room's damper.
- Compressor reversal cooldown: **3** (minutes)
- Last-changeover helper: `input_datetime.zc_last_changeover`
- Status / decision output *(optional but recommended)*: `input_text.zc_status`
  — the coordinator writes a one-line summary of every decision here, e.g.
  `Cooling: Master bedroom · set 68° (hall 70°) · Home Night`. Add it to a dashboard
  (or just watch it in Developer Tools → States) to see exactly what the
  coordinator did and why on each run. See **Reading the status line** below.
- Full manual override / bypass *(optional)*: `input_boolean.zc_bypass` — while on,
  the coordinator touches nothing. **Use this same helper in every Room Zone too.**
- House mode selector *(optional)*: `input_select.<your_house_mode>` — when it
  reads **Away** the unit is forced into the eco preset (Eco mode); when it reads
  **Vacation** the unit is turned off. Any other value falls back to the
  presence logic below.
  - House mode "Away" value: **Away** · House mode "Vacation" value: **Vacation**
    *(match your selector's exact option labels)*
- Home occupied toggle: `input_boolean.<your_home_occupied>`
- Wide presence zone: `zone.home_wide`
- Away action: **Set eco preset**
- Away preset name / Home preset name: match your unit's presets (defaults
  `eco` / `none`).

> **Eco Away caps each room at its own away setpoint.** When nobody is home with
> the eco action (or the house-mode selector is **Away**), the eco preset is
> applied and the coordinator keeps **serving rooms** — but each room uses its
> **away cool setpoint** (see each Room Zone's *Away Setpoint Override*), which you
> set warmer than its day target to save power while still keeping it from baking
> (e.g. the master at 75). This is per-room, so an unoccupied bedroom can't drift
> to 78 the way a hallway-only band allowed. Use **Vacation** to turn the unit
> fully off instead.
- Night starts / ends at: **21:00:00** / **07:00:00** *(label-only — drives the
  `Home Night` vs `Home Day` status; the window may cross midnight)*
- Night/sleep toggle *(optional)*: `input_boolean.<your_sleep_mode>` — while on,
  the status shows `Home Night` regardless of the clock. Changes no conditioning.
- **Idle Behavior** (what to do when no room is calling):
  - Idle behavior: **Hold a heat_cool safety band** — keeps the unit in
    `heat_cool` on the band below, so the whole house stays between the limits
    (a high *and* low backstop). Requires a unit that supports a `heat_cool` /
    auto range mode. The other options are **Turn the unit off** (original) and
    **Off, but force cooling above the high limit** (cooling-only cap).
  - High limit / band high: **76** — the `heat_cool` cool target (or, in cap
    mode, the temperature above which cooling is forced).
  - Band low: **64** — the `heat_cool` heat target (band mode only).
  - A room's own demand always takes priority over the idle backstop. **Band mode
    requires Setpoint Force-Run (below) to be enabled:** while idling the band
    uses the unit's native dual-setpoint control, but when a room starts calling
    the unit switches to a single mode and inherits the band's target (e.g. the
    high) — Force-Run is what overrides that so the unit actually runs for the
    room. (Cap mode likewise needs Force-Run on to run.)

> ⚠️ **Vacation turns the whole unit off**, which also stops the always-on
> server room. If the server room must keep cooling while you travel, use
> **Away** instead of Vacation. A house-mode change is applied on the next
> safety re-sync (within ~2 min), not instantly.
- **Away Circulation Purge** *(optional — mold / condensation prevention)*:
  - Run a circulation purge while away: **on** to enable (default off).
  - Purge-window helper: `input_datetime.zc_purge_until` — **required.** Point
    every Room Zone's *Circulation purge window* at this **same** helper.
  - Purge every: **8** (hours) · Purge duration: **10** (minutes).
  - Also purge during Vacation: **on** *(recommended — a vacation is the longest
    the ducts sit unused)*.
  - Fan-only mode name: **fan_only** *(change only if your unit names it
    differently)*.
  - While the house is away (or on vacation) and nothing is calling, the
    coordinator periodically runs the blower in **fan-only** with **every damper
    open**, moving air through all the ducts to discourage stagnant-air mold and
    condensation. It never heats or cools, and it **yields the unit instantly**
    the moment a room calls or you come home (the window ends and normal control
    resumes). The status line reads `Circulation purge - fan on, all dampers open
    (mold prevention) - <mode>`.
- **Setpoint Force-Run** (optional but recommended — see below):
  - Manage the unit's setpoint: **on**
  - Force-run offset: **2** · Setpoint floor: **60** · Setpoint ceiling: **85**
  - Manual setpoint override toggle: `input_boolean.zc_setpoint_manual`
  - **Enable auto manual-edit grace: off** *(recommended — see note)*
  - Manual-edit grace / last-commanded / grace-until helpers: only needed if you
    turn the grace on; leave them unset otherwise.

> **Theater guest room settings moved.** The coordinator no longer has a
> *Theater Guest Room* section — guest mode, setpoints, sleep, pre-cool and boost
> all live on the theater's own Room Zone (see **Theater / bonus** above). If your
> existing coordinator automation still has the old guest inputs saved, they're
> simply ignored; you can clear them when you next edit it.

> **The auto manual-edit grace is OFF by default — keep it off.** It tried to
> auto-detect a by-hand setpoint change and back off, but it proved too sensitive
> to thermostat quirks (stepping, clamping, °C/°F rounding) and could get stuck
> reporting "recent manual change - leaving thermostat alone." With
> **Enable auto manual-edit grace = off** (default), force-run always manages the
> setpoint and this can never happen. For deliberate manual control, use the hard
> `zc_setpoint_manual` toggle instead — that never gets stuck. (Only turn the
> grace on if you specifically want hand-edit detection and accept the risk; it
> also needs the last-commanded and grace-until helpers set.)

> **Guest room mode** turns the theater from a passive relief valve into a
> conditioned bedroom. While the toggle is on and the theater is above its cool
> setpoint it calls for cooling like any room and its damper opens to serve it
> (after the nursery and master); while off (or not calling) the theater keeps
> working as the pressure-relief valve. Like every room, it needs Setpoint
> Force-Run on for the unit to actually run for it.

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

**Keeping manual control.** Three levels, lightest to heaviest:
- **Setpoint only** — turn `input_boolean.zc_setpoint_manual` **on**. The
  coordinator stops touching the *setpoint* (you pick the temperature) but still
  runs on/off, mode, and the dampers. Off = hand setpoint control back.
- **Full manual override / bypass** — turn `input_boolean.zc_bypass` **on**. The
  coordinator **and every room stand down completely**: no on/off, no mode, no
  setpoint, no damper moves. You have total manual control of the thermostat and
  all dampers until you turn it off, at which point automation resumes (within
  the ~2 min re-sync). **Point every Room Zone's *Full manual override* input and
  the Coordinator's at this same `zc_bypass` toggle.** The status lines read
  `Manual override ON ...` while it's active.
- **Soft (only if you enabled the off-by-default grace)** — change the setpoint by
  hand and the coordinator leaves it alone for the grace period. Off by default;
  prefer the toggles above.

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
   - **Priority yield (extreme only).** With the master's *Priority Yield* pointed
     at `input_text.nursery_urgent`: push the nursery **more than 2° past** its
     cool setpoint. `nursery_urgent` → `cool`, and while the nursery is served the
     master's dampers go **on** (closed) so airflow focuses on the nursery. As the
     nursery recovers to within ~2° (`nursery_urgent` → `none`), the master's
     dampers reopen and both share — even though the nursery is still cooling.
4. **Compressor reversal cooldown.** Watch `input_datetime.zc_last_changeover` —
   after a direction change, the unit won't reverse heat↔cool again until
   `min_changeover` minutes pass (unless no room is still calling the current
   direction). With the default of 3 min, the nursery is served ~3 min after the
   last reversal at worst; set the cooldown to 0 for instant reversal.
5. **Pressure relief while cooling.** With the server's *Cooling Ride-Along* on
   and its gate pointed at `input_boolean.zc_cool_relief`:
   - Make **both** the nursery and master call for cool. Two rooms are open, so
     `zc_cool_relief` stays **off**, the **server stays closed** (its status reads
     `Idle …`, not riding along), and all the air goes to the two bedrooms.
   - Now satisfy the master (or the nursery) so only **one** room is calling.
     Within a re-sync `zc_cool_relief` flips **on**, the **server damper opens**
     (status `Riding along (pressure relief) …`) as the second zone, and
     `switch.damper_theater` stays **on** (closed).
   - Force the server cold (below its ride-along floor) so it drops out, leaving
     one zone — now the theater goes **off** (open) as the backstop.
   (While heating, ride-along doesn't apply and the theater only opens when every
   room is closed.)
   - **Day, master yields → server relieves (not theater).** In the daytime, push
     the nursery >2° over so the master yields (its damper closes). Only the
     nursery is open, so `zc_cool_relief` turns **on** and the **server** opens as
     the relief — `switch.damper_theater` stays **on** (closed).
   - **Night → master is the relief until satisfied, then server.** With the
     master on its sleep/pre-cool setpoint (night) and *Only yield during the day*
     on: make the nursery call. The **master stays open** (it no longer yields) as
     the relief while it cools toward its own target — server closed. Let the
     master reach its target (it stops calling, damper closes): `zc_cool_relief`
     turns **on** and the **server** takes over as the relief.
   - A genuinely hot server (over its setpoint) always opens on its own,
     regardless of any of the above.
   - **Theater relief (guest mode off).** With guest mode off, close every room
     damper (nothing calling): `input_boolean.zc_theater_relief` turns **on** and
     the theater's status reads `Relief valve open (pressure relief)` with
     `switch.damper_theater` **off** (open). Make a room call so 2 zones open → the
     flag turns **off** and the theater closes (status `Off (disabled)`).
   - **Guest room mode.** Turn on `input_boolean.theater_guest_mode` and set the
     theater above its cool setpoint: `input_text.theater_demand` goes `cool`, the
     unit runs cool, and `switch.damper_theater` goes **off** (open) to serve it.
     Now make the master call: the theater's status reads `Cooling (yielding to
     priority) …` and its damper closes until the master is satisfied.
   - **Guest pre-cool + boost.** With guest mode on, after 20:00 (house occupied)
     the theater's target drops to its sleep setpoint (status `… (sleep)`). During
     the day, `input_boolean.theater_boost` does the same on demand (`… (boost)`).
6. **Main on/off.** With a room calling, the unit switches on in the winning
   direction. With every room satisfied/disabled, it switches off.
7. **Sleep / pre-cool.** Turn on `input_boolean.<your_sleep_mode>`; the master
   bedroom uses the 68 °F sleep cool setpoint. The same 68 °F target applies
   automatically inside the 20:00–07:00 window (house occupied) without touching
   the toggle — so an evening reading above 68 begins pre-cooling on its own.
   - **Comfort boost.** In the middle of the day, turn on
     `input_boolean.master_boost`: the master switches to the 68 °F target and
     becomes eligible even if the occupancy sensor reads clear, so a 73° room
     starts cooling; its status shows `Cooling … (boost)`. Turn it off → back to
     the 75 °F day target.
8. **Away.** Turn off `input_boolean.<your_home_occupied>` with nobody in
   `zone.home_wide` (or set the house-mode selector to Away). The eco preset is
   applied and each room switches to its **away** cool setpoint. With the master's
   away setpoint at 75, warm it past 75 → its status shows `Cooling … (away)`, it's
   served (even though unoccupied), and it's held at ~75 instead of drifting to 78.
   Rooms without an away setpoint just use their normal setpoints. Mode = `Eco Away`.
9. **Idle band.** With every room satisfied (nobody calling) and idle behavior
   set to **hold band**, the unit should sit in `heat_cool` on the 64–76 band
   (`Idle - holding 64-76° band` in the status line) — cooling if the house
   exceeds 76, heating below 64, otherwise coasting. (In **cap** mode instead, it
   stays off until the hallway passes 76, then shows
   `Starting cooling (hallway over 76°)`.)
10. **House mode.** Set the house-mode selector to **Away** → within ~2 min the
   unit switches to the eco preset but keeps running. Set it to **Vacation** →
   the unit turns off entirely (server room included). Set it back to **Home** →
   the eco preset is restored and normal operation resumes.
11. **Away circulation purge** *(if enabled).** With the purge on and the house
   away/vacation and nothing calling, the unit should periodically switch to
   `fan_only` with **every damper open** — each room's status reads `Circulation
   purge - damper open, fan circulating …` and the coordinator shows `Circulation
   purge - fan on, all dampers open (mold prevention)`. To test on demand, set
   `input_datetime.zc_purge_until` a few minutes into the future by hand — the fan
   runs and dampers open until it passes, then normal control resumes. Make a room
   call mid-purge (or come home) and the purge ends immediately, handing the unit
   back. (To time the cadence without waiting, temporarily lower *Purge every* to
   1 hour.)
9. **Force-run.** With force-run enabled, set the central thermostat to a mild
   value (e.g. 72) while a room calls for cooling and the hallway is cooler than
   that. The coordinator should drive the target down (≈ sensed − 2) so the unit
   runs, and `input_number.zc_last_cmd_setpoint` updates to that value.
10. **Setpoint override.** Turn `input_boolean.zc_setpoint_manual` on: the
    coordinator stops changing the setpoint (you set it) but still zones. Off →
    it resumes force-run.
11. **Full bypass.** Turn `input_boolean.zc_bypass` on: within the re-sync the
    status lines read `Manual override ON ...`, and now change the thermostat,
    mode, and any damper by hand — nothing gets reverted. Turn it off → automation
    resumes and takes the dampers/unit back over.

### Template sanity check

In **Developer Tools → Template**, paste (substituting your entity IDs) to
confirm the coordinator's core expressions:

```jinja
{% set zone_demands = ['input_text.nursery_demand','input_text.master_demand','input_text.server_demand'] %}
{% set zone_dampers = ['switch.damper_nursery','switch.damper_master_bedroom_1','switch.damper_master_bedroom_2','switch.damper_server_room'] %}
winner_dir:        {% set ns = namespace(dir='none') %}{% for d in zone_demands %}{% if ns.dir=='none' and states(d) in ['heat','cool'] %}{% set ns.dir=states(d) %}{% endif %}{% endfor %}{{ ns.dir }}
all_others_closed: {{ zone_dampers | reject('is_state','off') | list | count == (zone_dampers | count) }}
```

### Reading the status line

If you wired up the optional `input_text.zc_status` helper, the coordinator
writes a compact, scannable one-liner of what it decided. It **updates only when
the decision changes** (not every cycle), so use the entity's *last changed* time
to see when the coordinator last acted. Format — three parts joined by ` · `:

```
<action + room(s)> · <setpoint> · <house mode>
```

The **setpoint** part only appears when Setpoint Force-Run is actively managing
the target.

**Action + room(s)** — the action, with the room(s) it's serving merged in (no
separate "cooling:" list):
- `Cooling: Master bedroom` / `Heating: Nursery` — running in that direction for
  the named room(s); multiple rooms are comma-separated.
- `Cooling · hallway over 76°` — cooling only because the hallway passed the cap
  (no room asked), so there's no room name.
- `Waiting to cool · cooldown 1/3m: Nursery` — a reversal is wanted for the
  nursery but the compressor cooldown is blocking it (1 of 3 min elapsed).
- `Idle · 64-76° band` — nothing is calling and idle behavior is "hold band".
- `Off · no demand` — nothing is calling and idle behavior is "off".
- `Off` — the unit is forced off (see the mode: `Vacation`, or `Away` with the
  turn-off action).

**Setpoint** (force-run):
- `set 68° (hall 70°)` — the coordinator pushed the thermostat target to **68**
  while the hallway (which the thermostat senses) reads **70**. This is a *control
  lever* to keep the unit running — **not** the temperature any room is aimed at.
  Each room is served by its own damper; "68° while the hallway reads 70°" just
  means "keep cooling", it does **not** mean the house is being cooled to 68.
- `manual setpoint` — the hard manual override toggle is on; the coordinator isn't
  touching the target.
- `manual change - holding` — only if you enabled the (off-by-default) auto
  manual-edit grace.

**House mode:** `Home Day` · `Home Night` (inside the night window, or the
night/sleep toggle is on) · `Eco Away` (nobody home, eco action) · `Away` (nobody
home, "do nothing"/"turn off" action) · `Vacation` (house-mode selector). The
Day/Night split comes from the **Night starts/ends at** times (default 21:00 →
07:00) plus the optional **Night/sleep toggle** — both label-only.

Examples:

```
Cooling: Master bedroom · set 68° (hall 70°) · Home Night
Cooling: Nursery, Master bedroom · set 68° (hall 71°) · Eco Away
Waiting to cool · cooldown 1/3m: Nursery · Home Day
Idle · 64-76° band · Home Night
Off · no demand · Home Day
Off · Vacation
Circulation purge - fan on, all dampers open (mold prevention) - Away
```

If the status line **never updates**, the coordinator automation itself isn't
running — check that it's enabled and look at its trace. If it shows
`Off · no demand` while a room is hot, that room isn't publishing demand
(check its demand `input_text` and temp sensor).

### Reading a room's status

Each Room Zone can also write a plain-English line to its own status helper
explaining **why it is or isn't calling** — its temperature vs its *effective*
target, whether it's on day or sleep setpoints, and whether it's yielding or
disabled. Examples:

```
Cooling - 76.4° vs 75.0° target (day)                 ← above the 75° day target, so it's calling
Cooling - 68.5° vs 68.0° target (sleep)               ← on the sleep setpoint (why it cools a ~68° room)
Cooling - 76.0° vs 75.0° target (away)                ← nobody home: held at the away setpoint (not baking)
Idle - 71.0° (cool above 72.0°, heat below 68.0°) (day)  ← comfortable, not calling
Heating - 66.5° vs 68.0° target
Cooling (yielding to priority) - 76.0° vs 75.0° target (day)  ← damper closed to focus air on the nursery
Riding along (pressure relief) - 72.0°, taking cool air down to 70.0°  ← server: 2nd zone while cooling
Idle - 73.0° (not eligible: occupancy/schedule)       ← gated off by occupancy or its window
Off (disabled)                                        ← enable toggle is off
No temperature reading                                ← sensor unavailable (fails safe to idle)
```

This is the companion to the coordinator line: the coordinator explains what the
*unit* is doing; each room status explains what that *room* wants and why. The
`(day)` / `(sleep)` / `(away)` tag is the quickest way to catch a stuck sleep
toggle or confirm the away cap is in effect.

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
