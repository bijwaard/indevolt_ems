# Indevolt EMS for Home Assistant

Use surplus PV power to charge an Indevolt battery, discharge it to reduce grid
import, and optionally switch an electric water heater when sufficient surplus is available.
The controller uses a configurable power limit, proportional adjustment and a
minimum target difference to reduce small control updates.

This repository contains a working installation's YAML configuration. It is a
reference configuration, **not an importable blueprint or a complete, portable
Home Assistant installation**. Replace the installation-specific entities and
device IDs before enabling its automations.

## Files

| File | Purpose |
| --- | --- |
| [configuration.yaml](configuration.yaml) | Helpers, template sensors, power-limit scripts and installation-specific configuration |
| [automations.yaml](automations.yaml) | Battery control, helper reset and optional boiler automations |
| [battery_panel.yaml](battery_panel.yaml) | Full dashboard definition with monitoring and controls |
| [mqtt_without_smartmeter.yaml](mqtt_without_smartmeter.yaml) | Optional Domoticz MQTT PV sensors; not a replacement for the controller's grid-power inputs |
| [mqtt_with_smartmeter.yaml](mqtt_with_smartmeter.yaml) | Optional Domoticz MQTT PV sensors and smart-meter as replacement for the controller's grid-power inputs |
| [examples/](examples/README.md) | Portable helper, meter and dashboard snippets with installation instructions |

Note that MQTT was initially used for my grid power read by Domoticz, 
the grid measurements from the P1 meter are now shared between Domoticz and Home Assistant using ser2net and socat.

## Requirements

- Home Assistant supporting the YAML syntax used in these files. No minimum
  supported version has been established.
- An installed Indevolt integration providing `indevolt.charge`,
  `indevolt.discharge`, a standby button, battery SOC and real-time mode.
  The integration itself is not included here. Check its action fields in
  **Developer Tools > Actions**; this configuration uses `device_id`,
  `target_soc` and `power` in watts.
- Current grid import/export power, either as two non-negative sensors or as
  one signed sensor. Energy totals in kWh cannot substitute for power readings.
- For boiler control: a controllable plug and, preferably, a power sensor to
  distinguish a requested switch state from actual heating.

## Installation

1. Back up your Home Assistant configuration. Keep this controller disabled
   while mapping entities and checking the templates.
2. Install/configure the Indevolt integration. In **Developer Tools > States**,
   identify the battery SOC, real-time mode, AC input/output power and standby
   button. Replace all `indevolt_cms_sf2000` references with your actual entities.
   Confirm that real-time mode reports `charging`, `discharging` and `standby`,
   or adapt the `Battery Virtual State` template to your integration's values.
3. Replace the battery device ID `c9489c0a5a20c9227366b0680f81b482` in both
   `indevolt` actions. This is an installation-specific device registry ID,
   not the battery model or serial number. Select your device in the action
   editor and inspect the resulting YAML to obtain the correct ID.
4. Merge [examples/helpers.yaml](examples/helpers.yaml) into your existing
   configuration. It includes `input_boolean.boiler_block_battery`, which is
   referenced by the controller but is **not defined in the root configuration
   export**. Keep that helper off when you do not use boiler control.
5. Configure exactly one `sensor.power_surplus`. Use the root template for
   separate import/export sensors in kW, or adapt one of the
   [meter examples](examples/README.md#meter-inputs). Do not install multiple
   definitions for the same entity.
6. Copy the root configuration's `Battery Virtual State` and `battery_surplus`
   template sensors into your `template:` configuration. The measured battery
   power template is useful for monitoring, but is not used as the controller's
   remembered power. Check actual entity IDs after loading: existing registry
   entries may retain an older entity ID despite a `default_entity_id`.
7. Merge `script battery:` from the root configuration if you want the
   configurable increase/decrease buttons. Preserve your existing
   `script: !include scripts.yaml`. Home Assistant supports separately labelled
   script sections; see [splitting configuration](https://www.home-assistant.io/docs/configuration/splitting_configuration/).
8. Add **battery control** and **battery reset memory on static** from
   `automations.yaml` to your own automation list. Do not overwrite existing
   automations. Keep IDs unique and avoid installing a second copy.
9. If using the boiler, also add **Boiler on**, **Boiler off** and
   **Boiler Safety Block Turn Off**. Replace boiler device ID
   `ef9813aa9e883f924d40150519fe6d74`, or replace its action targets with your
   explicit `switch.<your_boiler>` entity. Otherwise omit those three
   automations and keep the block helper off.
10. Check the configuration with Home Assistant's configuration check before
    restarting/reloading. Inspect the new sensors and helpers before enabling
    battery control. Disable old **battery charging**, **battery discharge**,
    **battery idle**, charging/discharging reset automations, or other
    controllers that issue competing commands. Keep the new helper-reset
    automation enabled.
11. Add the dashboard. `battery_panel.yaml` is a **whole dashboard** (`views:`),
    not a single card. Use a new dashboard's raw configuration editor, or merge
    its view into your existing dashboard. Replace/remove unrelated entities.
    [examples/controls-card.yaml](examples/controls-card.yaml) is a smaller card
    that also exposes the configurable limit-step scripts.

Do not replace your whole `configuration.yaml` with the root example. It contains
local URLs, Zigbee settings, gas/energy meters, unrelated appliance sensors and
includes for `scripts.yaml`, `scenes.yaml` and `themes`, which are not provided.
Keep your own `default_config:` and existing integration settings. MQTT is optional;
the supplied MQTT file alone does not provide the grid meter entities expected
by `sensor.power_surplus`.

## Sign conventions and units

| Value | Unit | Positive | Negative |
| --- | --- | --- | --- |
| `sensor.power_surplus` | kW | Net export to the grid | Net import from the grid |
| `input_number.real_time_battery_power` | kW | Remembered charging setpoint | Remembered discharging setpoint |
| `sensor.battery_surplus` | kW | Estimated surplus before battery compensation | Estimated deficit before battery compensation |
| Measured battery power template | W | AC input exceeds AC output | AC output exceeds AC input |
| `indevolt.charge` / `indevolt.discharge` action `power` | W | Positive magnitude for either action | Never send a negative action magnitude |

The templates use:

```text
power_surplus = grid_export_kW - grid_import_kW
battery_surplus = power_surplus + remembered_battery_power_kW
measured_battery_power_W = battery_AC_input_W - battery_AC_output_W
```

For example, exporting 0.2 kW while charging at a remembered 1.5 kW gives an
estimated 1.7 kW of surplus before battery compensation. Importing 0.1 kW while
discharging at a remembered -1.0 kW gives an estimated -1.1 kW deficit.

The helper is the **last requested setpoint**, not a measurement of actual
battery power. The surplus estimate assumes the battery follows that request.
Communication delays, losses, SOC constraints and rejected commands can make
that assumption temporarily wrong. The boiler remains part of the household
load in this estimate.

## How battery control works

The automation runs on changes to `sensor.power_surplus`, the power limit,
damping percentage or minimum difference. It is event-driven, not a fixed-rate
control loop. Meter update frequency therefore affects response speed.

1. In virtual `static` state, use grid surplus directly and treat remembered
   battery power as zero. The separate reset automation also clears the helper
   on transitions to `static` or `idle`.
2. Choose a target, in kW:
   - Force Cheap Energy on: positive power limit.
   - Surplus above 0.05 kW and SOC below 100%: charge, capped by the limit.
   - Surplus below -0.05 kW and SOC above 10%: discharge, capped by the limit.
   - Otherwise: target zero.
3. Smooth the request:

   ```text
   new_power = current + (target - current) * damping_percent / 100
   new_power = clamp(new_power, -power_limit_kW, +power_limit_kW)
   ```

4. For charging/discharging, issue a command only when the absolute difference
   between **target and current** exceeds the minimum difference. Charging also
   requires `boiler_block_battery` to be off; this block does not suppress the
   discharge or standby branches.
5. When the smoothed value is within ±0.05 kW and `current != 0`, clear the
   helper and request standby. The numeric action is rounded to watts.

With current 1.0 kW, target 2.0 kW and 30% damping, the next request is 1.3 kW.
At 50% it is 1.5 kW. With 50% damping and a 100 W minimum target difference,
a difference of exactly 100 W does not trigger an update (`>` is strict).
A difference of 120 W produces a 60 W adjustment, if other conditions pass.

The virtual-state sensor marks charging above 99% SOC, discharging at or below
10%, and standby as `static`. These thresholds and the action target SOCs
(100% / 10%) are separate literals; adapt them together if needed.

## Settings

| Helper (`input_number.` prefix) | Initial value | Meaning |
| --- | --- | --- |
| `battery_power_limit` | 2400 W | Common cap for charging and discharging; range 100–3000 W |
| `battery_damping_percent` | 30% | Fraction of the target error applied each update; range 1–100% |
| `battery_min_power_difference` | 100 W | Minimum absolute target/current difference; range 0–1000 W |
| `battery_power_limit_step` | 100 W | Increment used only by the increase/decrease scripts; range 1–1000 W |
| `real_time_battery_power` | Restored state, or helper default | Internal remembered setpoint, range -3 to +3 kW; do not use as a manual power command |

A larger damping percentage responds faster. A smaller percentage smooths the
response but follows changing cloud cover or loads more slowly. A larger minimum
difference suppresses small updates; it does not filter large cloud-related
power changes. Tune one setting at a time and compare grid power and actual
battery power under similar conditions.

The step helper does **not** affect automatic damping or the native helper's
100 W arrows. Use `script.battery_power_limit_increase` and
`script.battery_power_limit_decrease`: from 2400 W, a 50 W step gives 2450/2350 W,
whereas a 200 W step gives 2600/2200 W. Scripts clamp the result to helper bounds.

The four settings have explicit `initial:` values and return to those values
on Home Assistant startup. Edit `initial:` for a permanent startup change, or
remove it to use restored state and initialize the helper deliberately before
running the controller. See [input number state restoration](https://www.home-assistant.io/integrations/input_number/#restore-state).

Charging at 3000 W has been observed on the author's installation. This does
not establish 3000 W discharge support or compatibility with every battery or
firmware. Select a limit supported by your installation.

## Optional boiler and cheap-energy control

- Boiler on: `battery_surplus > 1.8 kW` for 10 seconds, or Force Cheap Energy
  changing from off to on. Requires the block helper off and either sufficient
  surplus or Force Cheap Energy on.
- The on sequence sets the block helper, requests the plug on, waits 30 seconds,
  then clears the block.
- Boiler off: a deficit below -1.1 kW for 15 seconds, or Force Cheap Energy
  turning off. Both paths still require the deficit condition and the force
  helper off. Turning force off alone does not necessarily turn the boiler off.
- The block cleanup checks once per minute and at startup, plus after the block
  has been on for one minute. Its condition requires at least 45 seconds on.
  Its description still says 30 seconds; the actual YAML timings govern.

These are surplus-power rules, not water-temperature control. A switch reporting
on does not prove heating: inspect both switch state and measured power.
A numeric-state trigger is a threshold-crossing event, not a repeated command
while surplus stays high. Use automation traces when investigating a missed
switch action.

Force Cheap Energy is a manual/external flag, not a tariff scheduler. In the
current battery automation it is read at the next listed trigger; changing it
alone is not a battery-control trigger. It does directly trigger the boiler
on/off automations when those are installed.

## Current limitations and troubleshooting

- Missing/unknown input values are often converted with `float(0)`. This is not
  an explicit stale-data timeout or a guaranteed standby fallback. Verify meter
  availability and integration behavior before relying on unattended control.
- The remembered power helper is updated **before** the battery action. A failed
  action can leave memory different from actual power. Check action errors and
  compare the helper against measured AC input/output power.
- SOC, mode, Force Cheap Energy and boiler-block changes are not direct triggers
  for `battery control`; a later meter/settings update normally applies them.
- `mode: single` ignores a new trigger while a run is still executing; it does
  not queue every meter update. See [automation modes](https://www.home-assistant.io/docs/automation/modes/).
- An automation's top-level on/off state is visible in the automation list or
  its `automation.*` entity. `enabled: false` on one trigger does not disable
  the entire automation.
- For a boiler that fails to heat, inspect its switch history and the trace's
  resolved switch target. A block-helper pulse alone is not confirmation that
  the plug turned on.

Use Home Assistant's configuration check and automation traces for deployment
validation. Parsing YAML outside Home Assistant cannot verify device IDs,
entity availability, integration action schemas or physical battery behavior.

## Credits and licensing:

If you use or adapt this integration, please credit the original project and 
link to this repository.
