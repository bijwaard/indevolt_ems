# Configuration examples

These are mergeable snippets, not complete Home Assistant configurations or
blueprints. Replace placeholder meter entities and choose only one meter example.

## Helpers

Merge [helpers.yaml](helpers.yaml) into the existing `input_number:` and
`input_boolean:` mappings in `configuration.yaml`, adding entries rather than
duplicating keys. Do not create duplicate UI helpers with the same desired names.
This includes the missing boiler block helper; keep it off for battery-only use.
The four tuning helpers start at the supplied `initial` values on each restart.

## Meter inputs

Choose one of:

- [meter-import-export-w.yaml](meter-import-export-w.yaml): separate non-negative
  import/export power sensors reporting watts. Converts W to kW and subtracts
  import from export.
- [meter-signed-w.yaml](meter-signed-w.yaml): one signed grid-power sensor, where
  **positive means import**, reporting watts. Negates it and converts to kW.
  If your meter already reports export-positive power, remove the minus sign.

These files contain a top-level `template:` list. Merge the list entries into
an existing `template:` section, or use a labelled section. Do not place the
whole file under `template: !include ...` without first removing its top-level
`template:` key. Do not install either example alongside the root
`power_surplus` definition.

Replace `sensor.your_grid_import_power_w`, `sensor.your_grid_export_power_w`,
or `sensor.your_signed_grid_power_w` with actual entity IDs. Check units before
applying the `/ 1000` conversion. None of these templates uses cumulative
energy readings.

With 1200 W export and 200 W import, expect `sensor.power_surplus = 1.0 kW`.
With signed grid power of +500 W (import), expect -0.5 kW. With -1500 W
(export), expect +1.5 kW. Unknown/unavailable sources make the example sensor
unavailable, but the current controller's `float(0)` fallbacks still need to be
considered; this is not a complete data-loss shutdown mechanism.

The root MQTT file provides PV readings only. PV production alone is insufficient:
the controller needs household grid exchange or an equivalent net-power estimate.

## Controls

Add [controls-card.yaml](controls-card.yaml) as one manual dashboard card. Install
the helpers and copy the root configuration's `script battery:` block first.
The two script rows provide Run buttons which honor `battery_power_limit_step`.
The normal numeric input's native arrows retain the configured fixed step.
Changing `battery_power_limit_step` alone does not change the current limit.

For a manual settings change in **Developer Tools > Actions**:

```yaml
action: input_number.set_value
target:
  entity_id: input_number.battery_damping_percent
data:
  value: 50
```

For one configurable upward limit step:

```yaml
action: script.battery_power_limit_increase
```

These actions modify settings; enabling the controller can then cause battery
commands on the resulting settings updates. Check the current power limit and
entity mapping before enabling the automation.
