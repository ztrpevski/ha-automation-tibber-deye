# Tibber + Deye/Solarman Smart Arbitrage — Home Assistant Automation v9

Automatically **buy electricity cheaply** from the grid and **sell/export at peak prices** using a Deye/Solarman inverter and Tibber dynamic pricing, all inside Home Assistant.

## What it does

Every day at `00:30` and `15:30` (and on HA start) the automation:

1. Fetches the full 48-hour Tibber price forecast via the `tibber.get_prices` service.
2. Finds the **cheapest contiguous charging block** in the upcoming window and programs the inverter to grid-charge at that time.
3. Finds the **most expensive contiguous selling block** after the charge window and programs the inverter to sell/export to the grid at that time.
4. Writes a 6-period Time-of-Use (TOU) schedule directly to the Deye/Solarman inverter using the Solarman integration entities.
5. Guards against writing bad data: if no price slots are returned, the previous TOU schedule is kept unchanged.

## Requirements

| Requirement | Details |
|---|---|
| Home Assistant | 2024.x or newer |
| Tibber integration | [Tibber for HA](https://www.home-assistant.io/integrations/tibber/) with `tibber.get_prices` service |
| Solarman / Deye integration | Entities: `select.inverter_program_N_charging`, `number.inverter_program_N_power`, `number.inverter_program_N_soc`, `time.inverter_program_N_time` (N = 1-6) |
| Battery SOC sensor | `sensor.inverter_battery_capacity` (percentage) |
| Tibber price sensor | `sensor.YOUR_HOME_NAME_electricity_price` — **replace `YOUR_HOME_NAME`** with your actual entity name |

## Quick start

### Option A — all-in-one package (recommended)

1. Copy `packages/tibber_deye_smart_arbitrage_package.yaml` to `/config/packages/tibber_deye_smart_arbitrage_package.yaml`.
2. Ensure `configuration.yaml` contains:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. **Find and replace** `YOUR_HOME_NAME` with your Tibber sensor prefix (e.g. `my_house_42`).
4. Check configuration → Restart HA.

### Option B — split files

Include each file under its matching domain in `configuration.yaml`:

```yaml
automation: !include split_files/automations_tibber_deye_smart_v9_boundary_force_sell.yaml
template: !include split_files/templates_tibber_deye_v9_boundary_force_sell.yaml
input_boolean: !include split_files/input_booleans_tibber_deye_v9_boundary_force_sell.yaml
input_number: !include split_files/input_numbers_tibber_deye_v9_boundary_force_sell.yaml
script: !include split_files/scripts_tibber_deye_v9_boundary_force_sell.yaml
```

> **Note:** Do not use both the package and the split files at the same time.

## Personalisation — replace `YOUR_HOME_NAME`

Every reference to `sensor.YOUR_HOME_NAME_electricity_price` must point to your actual Tibber sensor.

Find it in HA under **Settings → Devices & Services → Tibber** or run this in the template editor:

```jinja
{{ states.sensor | selectattr('entity_id', 'search', 'electricity_price') | map(attribute='entity_id') | list }}
```

Then do a global find-and-replace:

```
YOUR_HOME_NAME  →  your_actual_home_slug
```

across all YAML files in this bundle.

## Inverter entity naming

This bundle assumes the standard Solarman/Deye entity format:

```
select.inverter_program_1_charging   (options: Grid, Disabled)
number.inverter_program_1_power      (W)
number.inverter_program_1_soc        (%)
time.inverter_program_1_time         (HH:MM:SS)
sensor.inverter_battery_capacity     (%)
```

If your entities are named differently, replace them throughout the YAML files.

## Input helpers (created automatically by the package)

| Helper | Default | Purpose |
|---|---|---|
| `input_number.tibber_deye_battery_capacity_kwh` | `20.48` | Usable battery size |
| `input_number.tibber_deye_charge_power_w` | `5000` | Grid charge power |
| `input_number.tibber_deye_sell_power_w` | `10000` | Export/sell power |
| `input_number.tibber_deye_night_power_w` | `1000` | Night self-use power limit |
| `input_number.tibber_deye_charge_efficiency` | `0.92` | Charge round-trip efficiency |
| `input_number.tibber_deye_discharge_efficiency` | `0.92` | Discharge efficiency |
| `input_number.tibber_deye_charge_target_soc` | `90` | Target SOC after charging |
| `input_number.tibber_deye_sell_stop_soc` | `25` | Stop selling at this SOC |
| `input_number.tibber_deye_night_stop_soc` | `20` | Night self-use stops at this SOC |
| `input_number.tibber_deye_neutral_soc` | `25` | SOC for neutral/idle periods |
| `input_number.tibber_deye_min_price_spread_eur_kwh` | `0.08` | Minimum spread required to enable arbitrage |
| `input_number.tibber_deye_emergency_soc_below` | `18` | Trigger emergency charge below this SOC |
| `input_number.tibber_deye_emergency_charge_target_soc` | `40` | Emergency charge target |
| `input_number.tibber_deye_emergency_charge_power_w` | `4000` | Emergency charge power |
| `input_boolean.tibber_deye_force_sell_export` | `on` | Force sell even if spread guard would block it |
| `input_boolean.tibber_deye_debug_notifications` | `off` | Show detailed schedule notification after each run |

## How the TOU schedule is built

Deye/Solarman TOU works with 6 programs, each with a **start time** that acts as the end time of the previous program. v9 sorts all six period boundaries and writes them in ascending order to avoid duplicate `00:00` boundaries.

```
Period 1 (night_start):    00:00  Disabled  1000 W  SOC 20%   — night self-use
Period 2 (charge_start):   HH:MM  Grid      5000 W  SOC 90%   — cheapest window
Period 3 (charge_end):     HH:MM  Disabled     0 W  SOC 25%   — after charge, neutral
Period 4 (night_end):      06:00  Disabled     0 W  SOC 25%   — end of morning window
Period 5 (sell_start):     HH:MM  Disabled* 10000 W  SOC 25%  — peak price export
Period 6 (sell_end):       HH:MM  Disabled     0 W  SOC 25%   — back to neutral
```

\* The Deye "sell/export" mode is labelled **Disabled** in the Solarman integration when battery-first selling is enabled at the inverter level. If your inverter shows a different option name (e.g. "Selling First"), the automation auto-detects it from the select entity options.

## Price spread guard

Arbitrage is only activated when:

```
sell_average_price - charge_average_price >= min_price_spread_eur_kwh
```

Override this by setting `input_boolean.tibber_deye_force_sell_export` to `on`.

## Emergency low SOC

A separate automation (`tibber_deye_low_soc_emergency_charge`) watches `sensor.inverter_battery_capacity`. If it drops below `input_number.tibber_deye_emergency_soc_below` (default 18 %), it immediately starts a conservative grid charge regardless of the TOU schedule.

## Debug notifications

Turn on `input_boolean.tibber_deye_debug_notifications` to receive a Home Assistant persistent notification after each schedule run showing:

- Total slots, slot width (15 or 60 min)
- Chosen charge window and average price
- Chosen sell window and average price
- Spread and whether arbitrage is active
- Hour-by-hour mode (CHARGE / SELL / NEUTRAL) for the next 24 slots

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Schedule not updating | Automation condition fails | Check `sensor.YOUR_HOME_NAME_electricity_price` is not `unknown` |
| `price_count: 0` | Tibber service returned no slots | Check `tibber.get_prices` is available; verify Tibber API token |
| Duplicate `00:00` TOU boundary | Old automation version | Use v9 from this bundle |
| Wrong select option (e.g. lowercase `grid`) | Integration uses different option names | Check `select.inverter_program_1_charging` options in HA developer tools |
| Emergency charge not triggering | SOC sensor unavailable | Confirm `sensor.inverter_battery_capacity` is reachable |

## File structure

```
README.md                                              ← this file
configuration_include_examples_v9_boundary_force_sell.yaml
packages/
    tibber_deye_smart_arbitrage_package.yaml           ← all-in-one, use this OR split files
split_files/
    automations_tibber_deye_smart_v9_boundary_force_sell.yaml
    input_booleans_tibber_deye_v9_boundary_force_sell.yaml
    input_numbers_tibber_deye_v9_boundary_force_sell.yaml
    scripts_tibber_deye_v9_boundary_force_sell.yaml
    templates_tibber_deye_v9_boundary_force_sell.yaml
checks/
    validation_tibber_deye_v9_boundary_force_sell.json
docs/
    README_Tibber_Deye_v9_Boundary_Force_Sell.md       ← legacy doc
```

## License

MIT — free to use, modify, and share.
