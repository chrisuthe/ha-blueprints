# Room Climate Orchestration (Phase 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the Room Climate Orchestrator blueprint and roll it out to all four zones of the Sioux Falls house, replacing the four drifted minisplit automations, both warm-ups, and the `use_heat_pumps` toggle.

**Architecture:** One HA blueprint (wall-thermostat-as-dial → minisplit mode/setpoint arbitration by room + outdoor temp) in public repo `chrisuthe/ha-blueprints`, imported by raw URL. Helpers, blueprint instances, and the warm-up schedule are created through the HA REST config API — no config-file access. Pilot on Will's room, then roll the rest.

**Tech Stack:** Home Assistant blueprint YAML (2024.10+ trigger/action syntax), HA REST API via PowerShell `Invoke-RestMethod`, git + gh CLI.

## Global Constraints

- HA base URL: `https://ha.home.chrisuthe.com`; token: `Get-Content "$env:USERPROFILE\.hass_token"` (never echo it into output).
- Every REST call uses header `Authorization: Bearer <token>`.
- Commits: plain messages, **no AI attribution, no Co-Authored-By lines** (user's global CLAUDE.md).
- Push before asking the user to test anything.
- All temperatures °F. Minisplit setpoint clamp: 62.5–86.
- Do not delete any existing automation until the validation window passes (Task 10); disable only.
- Spec: `docs/specs/2026-07-21-room-climate-orchestration-design.md`. Deviations locked in during planning:
  - Anti-flap is implemented as idempotence guard + mode-hold hysteresis instead of trigger persistence timers (same intent, more robust with numeric attribute triggers).
  - `boost_helper` is a **required** input; a 4th (unused-by-schedule) helper is created for Main Floor.
  - `lockout_switch` input exists but is **inert** in v1; drive logic lands with the Phase 2 blueprint update.
  - Initial dials = current minisplit targets (main 69, master 69, kids 70), not winter comfort numbers.
  - **v1.2 amendments (2026-07-21, mid-execution):** boost generalized to offset (−5…+5, `input_number.climate_offset_*`); blueprint inputs are now `offset_helper` + required `status_helper` (`input_text.climate_status_*`); night-setback `schedule.night_setback_*` helpers exist; Task 9's single warm-up automation is replaced by per-room "Climate Offset" glue automations (warm-up branch takes precedence over setback; Will's already exists as `climate_offset_will`). Gotcha: new `input_number` helpers initialize to their **minimum** — explicitly set 0 after creation.

## Entity Reference (used by every task)

| Zone | Dial | Minisplit(s) | Boost helper | Old automation id |
|---|---|---|---|---|
| Will's Room | `climate.zen_within_zen_01_9cca1200_fan_thermostat` | `climate.air_coniditioning_will_wills_bedroom` | `input_number.climate_offset_will` | `1711133709451` |
| Margaret's Room | `climate.zen_within_zen_01_1d6f1200_fan_thermostat` | `climate.air_conditioner_margaret_margaret_air_conditioner` | `input_number.climate_offset_margaret` | `1711134903934` |
| Master Bedroom | `climate.master_bedroom_thermostat_fan_thermostat` | `climate.master_bedroom_climate` | `input_number.climate_offset_master` | `1713543049333` |
| Main Floor | `climate.living_room_livingroomthermostat_climate` | `climate.main_floor_east`, `climate.main_floor_west` | `input_number.climate_offset_main_floor` | `1701969294983` |

Old warm-ups: `1644464608743` (master), `1639334415698` (kids). Toggle: `input_boolean.use_heat_pumps`.

---

### Task 1: Author the blueprint

**Files:**
- Create: `blueprints/automation/room_climate_orchestrator.yaml`
- Modify: `docs/specs/2026-07-21-room-climate-orchestration-design.md` (already amended: initial-dial correction)

**Interfaces:**
- Produces: blueprint importable at raw URL; once imported its `use_blueprint.path` is `chrisuthe/room_climate_orchestrator.yaml`. Input names consumed by Tasks 7/9: `dial_thermostat`, `minisplits` (list), `boost_helper`, `weather_entity`, `cool_band`, `cool_outdoor_floor`, `hp_offset`, `hp_outdoor_floor`, `hp_outdoor_max`, `calibration_offset`, `idle_mode`, `lockout_switch`.

- [ ] **Step 1: Write `blueprints/automation/room_climate_orchestrator.yaml`**

```yaml
blueprint:
  name: Room Climate Orchestrator
  description: >
    Treats a wall thermostat (kept in heat mode year-round) as the room's
    desired-temperature dial and drives one or more minisplit heads to satisfy it.
    Outdoor temperature picks the armed side: cooling at/above the cool floor,
    heat-pump heat inside the HP window, idle otherwise. Hot-water heat stays the
    hardwired backup via the wall thermostat itself. Room temperature is always read
    from the wall thermostat, never the minisplit head sensor.
  domain: automation
  source_url: https://github.com/chrisuthe/ha-blueprints/blob/main/blueprints/automation/room_climate_orchestrator.yaml
  input:
    dial_thermostat:
      name: Wall thermostat (dial)
      description: Family-facing thermostat. Its setpoint is the desired temperature; its sensor is the room temperature.
      selector:
        entity:
          domain: climate
    minisplits:
      name: Minisplit head(s)
      selector:
        entity:
          domain: climate
          multiple: true
    boost_helper:
      name: Boost helper
      description: input_number added to the heat entry threshold and heat target (morning warm-ups). Stays 0 when unused.
      selector:
        entity:
          domain: input_number
    weather_entity:
      name: Weather source
      default: weather.pirateweather
      selector:
        entity:
          domain: weather
    cool_band:
      name: Cooling entry band (°F above dial)
      default: 1.0
      selector:
        number:
          min: 0.5
          max: 5
          step: 0.5
          unit_of_measurement: "°F"
    cool_outdoor_floor:
      name: Minimum outdoor temperature for cooling (°F)
      default: 60
      selector:
        number:
          min: 40
          max: 80
          step: 1
    hp_offset:
      name: Heat target offset (°F above dial)
      default: 1.0
      selector:
        number:
          min: 0.5
          max: 3
          step: 0.5
          unit_of_measurement: "°F"
    hp_outdoor_floor:
      name: Minimum outdoor temperature for heat-pump heat (°F)
      default: 20
      selector:
        number:
          min: -20
          max: 50
          step: 1
    hp_outdoor_max:
      name: Maximum outdoor temperature for heat-pump heat (°F)
      description: Keep equal to the cooling floor so heat and cool are never armed together.
      default: 60
      selector:
        number:
          min: 40
          max: 80
          step: 1
    calibration_offset:
      name: Calibration offset (°F)
      description: Measured disagreement between the minisplit head sensor and the wall thermostat; added to the commanded heat target.
      default: 0
      selector:
        number:
          min: -5
          max: 5
          step: 0.5
          unit_of_measurement: "°F"
    idle_mode:
      name: Idle behavior
      default: "off"
      selector:
        select:
          options:
            - "off"
            - fan_only
    lockout_switch:
      name: Boiler lockout switch (Phase 2 — not driven yet)
      description: Reserved for the zone-panel interlock relay. Ignored by this version.
      default: []
      selector:
        entity:
          domain: switch
          multiple: true
mode: restart
max_exceeded: silent
triggers:
  - trigger: state
    entity_id: !input dial_thermostat
    attribute: temperature
  - trigger: state
    entity_id: !input dial_thermostat
    attribute: current_temperature
  - trigger: state
    entity_id: !input boost_helper
  - trigger: state
    entity_id: !input weather_entity
    attribute: temperature
  - trigger: state
    entity_id: !input minisplits
    to: unavailable
  - trigger: state
    entity_id: !input minisplits
    from: unavailable
  - trigger: homeassistant
    event: start
  - trigger: event
    event_type: automation_reloaded
conditions: []
actions:
  - variables:
      dial_entity: !input dial_thermostat
      minisplit_list: !input minisplits
      weather_ent: !input weather_entity
      boost_entity: !input boost_helper
      cool_band: !input cool_band
      cool_floor: !input cool_outdoor_floor
      hp_offset: !input hp_offset
      hp_floor: !input hp_outdoor_floor
      hp_max: !input hp_outdoor_max
      cal: !input calibration_offset
      idle_mode: !input idle_mode
      dial: "{{ state_attr(dial_entity, 'temperature') }}"
      room: "{{ state_attr(dial_entity, 'current_temperature') }}"
      outdoor: "{{ state_attr(weather_ent, 'temperature') }}"
      boost: "{{ states(boost_entity) | float(0) }}"
      head: "{{ minisplit_list if minisplit_list is string else minisplit_list[0] }}"
      head_mode: "{{ states(head) }}"
      head_target: "{{ state_attr(head, 'temperature') | float(0) }}"
      desired_mode: >-
        {%- if dial is none or room is none or outdoor is none -%} none
        {%- elif head_mode in ['unavailable', 'unknown'] -%} none
        {%- elif outdoor | float >= cool_floor | float -%}
          {%- if room | float >= dial | float + cool_band | float -%} cool
          {%- elif head_mode == 'cool' and room | float > dial | float - 0.5 -%} cool
          {%- else -%} idle
          {%- endif -%}
        {%- elif outdoor | float >= hp_floor | float and outdoor | float < hp_max | float -%}
          {%- if room | float <= dial | float + 0.5 + boost -%} heat
          {%- elif head_mode == 'heat' and room | float < dial | float + hp_offset | float + boost + 0.5 -%} heat
          {%- else -%} idle
          {%- endif -%}
        {%- else -%} idle
        {%- endif -%}
      desired_target: >-
        {%- if desired_mode == 'cool' -%}
          {{ [ [ dial | float - 0.5, 62.5 ] | max, 86 ] | min }}
        {%- elif desired_mode == 'heat' -%}
          {{ [ [ dial | float + hp_offset | float + boost + cal | float, 62.5 ] | max, 86 ] | min }}
        {%- else -%} 0
        {%- endif -%}
  - choose:
      - conditions: "{{ desired_mode == 'none' }}"
        sequence:
          - stop: Missing dial/room/outdoor data or minisplit unavailable.
      - conditions: "{{ desired_mode == 'cool' }}"
        sequence:
          - if: "{{ head_mode != 'cool' }}"
            then:
              - action: climate.set_hvac_mode
                target:
                  entity_id: !input minisplits
                data:
                  hvac_mode: cool
          - if: "{{ head_mode != 'cool' or (head_target - desired_target | float) | abs >= 0.25 }}"
            then:
              - action: climate.set_temperature
                target:
                  entity_id: !input minisplits
                data:
                  temperature: "{{ desired_target }}"
      - conditions: "{{ desired_mode == 'heat' }}"
        sequence:
          - if: "{{ head_mode != 'heat' }}"
            then:
              - action: climate.set_hvac_mode
                target:
                  entity_id: !input minisplits
                data:
                  hvac_mode: heat
          - if: "{{ head_mode != 'heat' or (head_target - desired_target | float) | abs >= 0.25 }}"
            then:
              - action: climate.set_temperature
                target:
                  entity_id: !input minisplits
                data:
                  temperature: "{{ desired_target }}"
    default:
      - if: "{{ head_mode != idle_mode }}"
        then:
          - action: climate.set_hvac_mode
            target:
              entity_id: !input minisplits
            data:
              hvac_mode: "{{ idle_mode }}"
```

- [ ] **Step 2: Validate the YAML parses**

Run: `python -c "import yaml; yaml.SafeLoader.add_constructor('!input', lambda l, n: l.construct_scalar(n)); yaml.safe_load(open(r'C:\CodeProjects\ha-blueprints\blueprints\automation\room_climate_orchestrator.yaml', encoding='utf-8')); print('OK')"`
Expected: `OK` (the custom constructor is needed because `!input` is an HA-specific tag).

- [ ] **Step 3: Commit**

```bash
git add blueprints/ docs/specs/
git commit -m "Add room climate orchestrator blueprint; correct initial dial values in spec"
```

---

### Task 2: Create the GitHub repo, README, push, verify raw URL

**Files:**
- Create: `README.md`

**Interfaces:**
- Produces: raw URL `https://raw.githubusercontent.com/chrisuthe/ha-blueprints/main/blueprints/automation/room_climate_orchestrator.yaml` (consumed by Task 4's import) and import helper link.

- [ ] **Step 1: Write `README.md`**

```markdown
# ha-blueprints

Home Assistant blueprints for the Sioux Falls house.

## Room Climate Orchestrator

Wall-thermostat-as-dial minisplit orchestration. Design doc:
[docs/specs/2026-07-21-room-climate-orchestration-design.md](docs/specs/2026-07-21-room-climate-orchestration-design.md)

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fchrisuthe%2Fha-blueprints%2Fmain%2Fblueprints%2Fautomation%2Froom_climate_orchestrator.yaml)
```

- [ ] **Step 2: Create the public repo and push**

Run: `gh repo create chrisuthe/ha-blueprints --public --source . --remote origin --push`
Expected: repo URL printed; push succeeds. (If `gh` isn't authenticated, run `gh auth status` and report to the user.)

- [ ] **Step 3: Commit README (if not included in creation push) and verify raw URL**

```bash
git add README.md && git commit -m "Add README with blueprint import link" && git push
curl -s -o /dev/null -w "%{http_code}" https://raw.githubusercontent.com/chrisuthe/ha-blueprints/main/blueprints/automation/room_climate_orchestrator.yaml
```

Expected: `200`.

---

### Task 3: Import the blueprint into HA (user step)

**Interfaces:**
- Produces: blueprint at HA path `chrisuthe/room_climate_orchestrator.yaml` (consumed by Tasks 7/9).

- [ ] **Step 1: Send the user the import link from Task 2's README and ask them to click Import in the HA dialog.**
- [ ] **Step 2: Verify import** — attempt a dry-run instance creation (Task 7 Step 2 body against id `roomclimate_probe`), expect either success (then immediately `DELETE /api/config/automation/config/roomclimate_probe`) or an error naming a missing blueprint path — if the path errors, list what HA imported by asking the user for the path shown in the UI, and adjust Tasks 7/9 accordingly.

---

### Task 4: One-time wall thermostat prep

**Interfaces:**
- Consumes: dial entity ids from the Entity Reference table.
- Produces: all four dials in `heat` mode with setpoints main 69, master 69, Margaret 70, Will 70.

- [ ] **Step 1: Set each dial to heat mode, then set its temperature** (PowerShell; repeat per row of the table below):

```powershell
$t = (Get-Content "$env:USERPROFILE\.hass_token" -Raw).Trim(); $h = @{ Authorization = "Bearer $t" }
$pairs = @(
  @{ e = 'climate.living_room_livingroomthermostat_climate'; temp = 69 },
  @{ e = 'climate.master_bedroom_thermostat_fan_thermostat'; temp = 69 },
  @{ e = 'climate.zen_within_zen_01_1d6f1200_fan_thermostat'; temp = 70 },
  @{ e = 'climate.zen_within_zen_01_9cca1200_fan_thermostat'; temp = 70 }
)
foreach ($p in $pairs) {
  Invoke-RestMethod -Method Post -Uri "https://ha.home.chrisuthe.com/api/services/climate/set_hvac_mode" -Headers $h -ContentType 'application/json' -Body (@{ entity_id = $p.e; hvac_mode = 'heat' } | ConvertTo-Json)
  Start-Sleep -Seconds 2
  Invoke-RestMethod -Method Post -Uri "https://ha.home.chrisuthe.com/api/services/climate/set_temperature" -Headers $h -ContentType 'application/json' -Body (@{ entity_id = $p.e; temperature = $p.temp } | ConvertTo-Json)
}
```

- [ ] **Step 2: Verify** — `GET /api/states/<entity>` for each: state `heat`, attribute `temperature` equals the target. Battery Z-Wave (Zens) may take a minute to report; re-check before declaring failure.

---

### Task 5: Create the four boost helpers

**Interfaces:**
- Produces: `input_number.climate_offset_will`, `_margaret`, `_master`, `_main_floor` (consumed by Tasks 7/9 and the schedule in Task 9).

- [ ] **Step 1: Create via REST** (repeat per id `climate_boost_will|margaret|master|main_floor`):

```powershell
$body = @{ name = 'Climate Boost Will'; min = 0; max = 5; step = 0.5; unit_of_measurement = '°F'; mode = 'slider' } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "https://ha.home.chrisuthe.com/api/config/input_number/config/climate_boost_will" -Headers $h -ContentType 'application/json' -Body $body
```

- [ ] **Step 2: Verify** — `GET /api/states/input_number.climate_offset_will` etc.: state `0.0`.

---

### Task 6: Disable Will's old automation

- [ ] **Step 1:** `POST /api/services/automation/turn_off` body `{"entity_id": "automation.minisplit_control_wills_room"}`.
- [ ] **Step 2: Verify** — `GET /api/states/automation.minisplit_control_wills_room` → state `off`.

---

### Task 7: Pilot instance — Will's room, with live cooling test

**Interfaces:**
- Consumes: blueprint path from Task 3, helper from Task 5.
- Produces: `automation.room_climate_will_s_room` (config id `roomclimate_will`).

- [ ] **Step 1: Create the instance**

```powershell
$body = @{
  id = 'roomclimate_will'
  alias = "Room Climate: Will's Room"
  use_blueprint = @{
    path = 'chrisuthe/room_climate_orchestrator.yaml'
    input = @{
      dial_thermostat = 'climate.zen_within_zen_01_9cca1200_fan_thermostat'
      minisplits = @('climate.air_coniditioning_will_wills_bedroom')
      boost_helper = 'input_number.climate_offset_will'
      idle_mode = 'fan_only'
    }
  }
} | ConvertTo-Json -Depth 10
Invoke-RestMethod -Method Post -Uri "https://ha.home.chrisuthe.com/api/config/automation/config/roomclimate_will" -Headers $h -ContentType 'application/json' -Body $body
```

Expected: `{"result": "ok"}`.

- [ ] **Step 2: Verify config readback** — `GET /api/config/automation/config/roomclimate_will` returns the blueprint instance.
- [ ] **Step 3: Live cooling-path test.** Read Will's room temp from the dial entity. Set the dial 1.5° *below* room temp via `climate.set_temperature` on the Zen (this is exactly what a family member turning the dial does). Within ~1 minute the minisplit must flip to `cool` with setpoint = dial − 0.5. Verify via `GET /api/states/climate.air_coniditioning_will_wills_bedroom`.
- [ ] **Step 4: Restore the dial to 70.** Confirm mode-hold: minisplit stays `cool` until the room reaches 69.5 (or flips to `fan_only` if already below).
- [ ] **Step 5: Check the automation trace** for errors: `GET /api/logbook/<ISO-now-minus-30min>?entity=automation.room_climate_will_s_room` — no errored runs.
- [ ] **Step 6: Commit any adjustments made** to the blueprint during the pilot (re-push + re-import if the YAML changed).

---

### Task 8: Observation gate (checkpoint — spans 1–2 days)

- [ ] **Step 1:** Report pilot status to the user; user watches Will's room comfort for 1–2 days.
- [ ] **Step 2:** Query the logbook daily for `automation.room_climate_will_s_room` runs and the minisplit's mode history (`GET /api/history/period/<start>?filter_entity_id=climate.air_coniditioning_will_wills_bedroom`); confirm no rapid mode flapping (mode changes should be minutes-to-hours apart, not seconds).
- [ ] **Step 3:** User explicitly approves rolling out to remaining rooms before Task 9 starts.

---

### Task 9: Roll remaining rooms + warm-up schedule

- [ ] **Step 1: Disable the three old automations** (`automation.control_minisplit_heat`, `automation.minisplit_automation_master_bedroom`, `automation.minisplit_control_margaret_s_room`) via `automation/turn_off`; verify each reads `off`.
- [ ] **Step 2: Create the three instances:**

```powershell
$instances = @(
  @{ id = 'roomclimate_margaret'; alias = "Room Climate: Margaret's Room"
     dial = 'climate.zen_within_zen_01_1d6f1200_fan_thermostat'
     splits = @('climate.air_conditioner_margaret_margaret_air_conditioner')
     boost = 'input_number.climate_offset_margaret'; idle = 'fan_only' },   # user preference post-rollout
  @{ id = 'roomclimate_master'; alias = 'Room Climate: Master Bedroom'
     dial = 'climate.master_bedroom_thermostat_fan_thermostat'
     splits = @('climate.master_bedroom_climate')
     boost = 'input_number.climate_offset_master'; idle = 'fan_only' },   # user preference post-rollout
  @{ id = 'roomclimate_main_floor'; alias = 'Room Climate: Main Floor'
     dial = 'climate.living_room_livingroomthermostat_climate'
     splits = @('climate.main_floor_east', 'climate.main_floor_west')
     boost = 'input_number.climate_offset_main_floor'; idle = 'fan_only' }
)
foreach ($i in $instances) {
  $body = @{
    id = $i.id
    alias = $i.alias
    use_blueprint = @{
      path = 'chrisuthe/room_climate_orchestrator.yaml'
      input = @{
        dial_thermostat = $i.dial
        minisplits = $i.splits
        offset_helper = $i.boost
        status_helper = ($i.boost -replace 'input_number\.climate_offset', 'input_text.climate_status')
        idle_mode = $i.idle
      }
    }
  } | ConvertTo-Json -Depth 10
  Invoke-RestMethod -Method Post -Uri "https://ha.home.chrisuthe.com/api/config/automation/config/$($i.id)" -Headers $h -ContentType 'application/json' -Body $body
}
```

Expected: three `{"result": "ok"}` responses.

- [ ] **Step 3: Verify** each instance's config readback and that each room's minisplit lands in a sensible state within 5 minutes (given July temps: `cool` or idle mode, setpoint tracking dial − 0.5 where cooling).
- [ ] **Step 4: Create the three remaining per-room offset glue automations** (Will's — `climate_offset_will` — already exists from the v1.2 pilot; these mirror it):

```powershell
$glues = @(
  @{ id = 'climate_offset_margaret'; alias = "Climate Offset: Margaret's Room"
     offset = 'input_number.climate_offset_margaret'; sched = 'schedule.night_setback_margaret'
     wu_start = '06:30:00'; wu_end = '07:30:00'; wu_val = 2; wu_weekdays_only = $true },
  @{ id = 'climate_offset_master'; alias = 'Climate Offset: Master Bedroom'
     offset = 'input_number.climate_offset_master'; sched = 'schedule.night_setback_master'
     wu_start = '05:30:00'; wu_end = '06:30:00'; wu_val = 3; wu_weekdays_only = $false },
  @{ id = 'climate_offset_main_floor'; alias = 'Climate Offset: Main Floor'
     offset = 'input_number.climate_offset_main_floor'; sched = 'schedule.night_setback_main_floor'
     wu_start = $null }
)
foreach ($g in $glues) {
  $triggers = @(
    @{ trigger = 'state'; entity_id = $g.sched },
    @{ trigger = 'homeassistant'; event = 'start' }
  )
  $branches = @()
  if ($g.wu_start) {
    $triggers += @{ trigger = 'time'; at = $g.wu_start }
    $triggers += @{ trigger = 'time'; at = $g.wu_end }
    $wuConds = @()
    $timeCond = @{ condition = 'time'; after = $g.wu_start; before = $g.wu_end }
    if ($g.wu_weekdays_only) { $timeCond.weekday = @('mon','tue','wed','thu','fri') }
    $wuConds += $timeCond
    $wuConds += @{ condition = 'numeric_state'; entity_id = 'weather.pirateweather'; attribute = 'temperature'; below = 45 }
    $branches += @{ conditions = $wuConds
      sequence = @(@{ action = 'input_number.set_value'; target = @{ entity_id = $g.offset }; data = @{ value = $g.wu_val } }) }
  }
  $branches += @{ conditions = @(@{ condition = 'state'; entity_id = $g.sched; state = 'on' })
    sequence = @(@{ action = 'input_number.set_value'; target = @{ entity_id = $g.offset }; data = @{ value = -3 } }) }
  $body = @{
    id = $g.id; alias = $g.alias; mode = 'restart'
    description = 'Writes the room climate offset: warm-up takes precedence over night setback.'
    triggers = $triggers; conditions = @()
    actions = @(@{ choose = $branches
      default = @(@{ action = 'input_number.set_value'; target = @{ entity_id = $g.offset }; data = @{ value = 0 } }) })
  } | ConvertTo-Json -Depth 15
  Invoke-RestMethod -Method Post -Uri "https://ha.home.chrisuthe.com/api/config/automation/config/$($g.id)" -Headers $h -ContentType 'application/json' -Body $body
}
```

Expected: three `{"result": "ok"}` responses. Verify each offset helper reads the correct value for the current time of day (0 outside setback/warm-up windows).

- [ ] **Step 5: Disable both old warm-ups** (`automation.warm_up_master_bedroom`, `automation.warm_up_the_kids`) and verify `off`.
- [ ] **Step 6: Smoke-test a boost:** set `input_number.climate_offset_will` to 2 via `input_number/set_value`, confirm the blueprint instance re-evaluates (logbook run), then set back to 0. (In July this exercises the trigger path; the heat branch stays un-armed — expected.)

---

### Task 10: Retirement (after user-approved validation window)

- [ ] **Step 1:** User confirms all four rooms have behaved through the validation window (suggest ≥3 days).
- [ ] **Step 2: Delete the six old automations** via `DELETE /api/config/automation/config/<id>` for ids `1711133709451`, `1711134903934`, `1713543049333`, `1701969294983`, `1644464608743`, `1639334415698`. Verify each subsequently 404s on GET.
- [ ] **Step 3: Delete the toggle** — `DELETE /api/config/input_boolean/config/use_heat_pumps`. If this errors, the boolean is YAML-defined: leave it and tell the user to remove it from `configuration.yaml` at leisure (it gates nothing once the old automations are gone).
- [ ] **Step 4: Update README** rollout status line; commit `"Mark Phase 1 rollout complete"` and push.

---

## Phase 2 (separate plan, when the Seeed relay board arrives)

Not covered here: ESPHome firmware for the XIAO ESP32-C6 (ALWAYS_OFF restore, API-disconnect release, 30-min auto-release), bench watchdog test, zone-panel wiring through COM+NC, blueprint v2 adding lockout drive logic + sag-release (room < dial − 1 for 20 min), re-import, fill `lockout_switch` per instance. Write that plan with the board in hand.
