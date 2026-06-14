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
| Deye/Solarman inverter | Supported: SE-3K, SE-5K, SE-10K, SG-3K, SG-5K, SG-10K and similar (with Time-of-Use slots) |
| Solarman WiFi dongle | **Required for communication** — A Solarman WiFi/Ethernet dongle (e.g., iE-DL-WIF002) must be connected to your Deye inverter to enable remote configuration via the Solarman app/cloud API |
| Solarman / Deye integration | [Solarman for HA](https://www.home-assistant.io/integrations/solarman/) integration installed and configured. Provides entities: `select.inverter_program_N_charging`, `number.inverter_program_N_power`, `number.inverter_program_N_soc`, `time.inverter_program_N_time` (N = 1-6) |
| Battery SOC sensor | `sensor.inverter_battery_capacity` (percentage) |
| Tibber price sensor | `sensor.YOUR_HOME_NAME_electricity_price` — **replace `YOUR_HOME_NAME`** with your actual entity name |

### WiFi Dongle Setup (Solarman)

The Deye inverter uses a **Solarman WiFi dongle** (hardware bridge) to communicate with the cloud and expose Time-of-Use programming entities in Home Assistant.

**Steps:**

1. **Install the dongle** — Connect the WiFi dongle to the RS485 port on your Deye inverter (usually a 2-pin terminal)
2. **Power it** — The dongle draws power from the inverter; no separate power supply needed
3. **Connect to WiFi** — Use the Solarman mobile app to add the dongle to your home WiFi network
4. **Pair in HA** — Install the [Solarman integration](https://www.home-assistant.io/integrations/solarman/), add a new device, and enter your dongle's IP address or serial number
5. **Verify entities** — Check **Settings → Devices & Services → Solarman** for 6 `program_N_charging` select entities (N = 1–6)

> **Note:** Without the WiFi dongle, you cannot write TOU schedules to the inverter remotely. This automation requires direct entity access to the 6 programming slots.

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

## How the 6 TOU Programming Slots Work

The Deye inverter has **exactly 6 programmable Time-of-Use (TOU) slots**, each controlling the inverter's behavior during a time window. Each slot is programmed via Home Assistant entity calls to the Solarman integration.

### Slot Structure

Each of the 6 slots is independently controlled by:

| Entity | Meaning | Example |
|---|---|---|
| `select.inverter_program_N_charging` | Operating mode | `Grid` (grid-charge) or `Disabled` (use battery or self-use) |
| `number.inverter_program_N_power` | Power setpoint | `5000` W for charging, `10000` W for selling |
| `number.inverter_program_N_soc` | Target State-of-Charge | `90%` after charge, `25%` after discharge |
| `time.inverter_program_N_time` | **Start time** of this slot | `14:30:00` (also marks end of previous slot) |

where **N = 1, 2, 3, 4, 5, 6**.

### How This Automation Builds the Schedule

Every 12 hours (00:30 and 15:30), the automation:

1. **Fetches 48-hour price forecast** from Tibber
2. **Identifies two windows:**
   - **Charge window:** Cheapest contiguous 1–4 hour block (grid-buy)
   - **Sell window:** Most expensive contiguous 1–4 hour block (export to grid)
3. **Calculates 6 boundaries** (start times):
   - Slot 1: `00:00` — Night self-use (1 kW limit)
   - Slot 2: `charge_start` — Grid charging begins
   - Slot 3: `charge_end` — Charging stops, back to neutral (0 W)
   - Slot 4: `06:00` — Morning boundary (adjustable)
   - Slot 5: `sell_start` — Peak export begins
   - Slot 6: `sell_end` — Back to neutral/disabled
4. **Sorts the 6 times** in ascending order (to avoid duplicate `00:00`)
5. **Writes each slot** via Solarman entity calls to the inverter

### Slot Configuration Example

```yaml
Period 1 (night_start):    00:00  Disabled  1000 W  SOC 20%   — night self-use (cheap hours)
Period 2 (charge_start):   02:15  Grid      5000 W  SOC 90%   — cheapest window (buy from grid)
Period 3 (charge_end):     04:45  Disabled     0 W  SOC 25%   — after charge, neutral
Period 4 (morning):        06:00  Disabled     0 W  SOC 25%   — end of night self-use
Period 5 (sell_start):     16:30  Disabled* 10000 W  SOC 25%  — peak price export (sell to grid)
Period 6 (sell_end):       19:00  Disabled     0 W  SOC 25%   — back to neutral
```

### Send to Inverter via Solarman WiFi Dongle

Once all 6 slots are calculated, the automation calls the Solarman integration to write each slot:

```yaml
service: select.select_option
target:
  entity_id: select.inverter_program_1_charging
data:
  option: Disabled

service: number.set_value
target:
  entity_id: number.inverter_program_1_soc
data:
  value: 20

service: time.set_value
target:
  entity_id: time.inverter_program_1_time
data:
  time: "00:00:00"
```

These calls flow through the **Solarman WiFi dongle → cloud API → inverter memory**, updating the inverter's TOU schedule in real-time.

> **Important:** The WiFi dongle must be powered and connected for the automation to successfully write the slots. If the dongle is offline, the entity calls will fail silently and the previous TOU schedule remains unchanged.

### Adaptive Mode Detection

The automation auto-detects the correct **sell/export mode** label:
- If `select.inverter_program_N_charging` options include `"Selling First"` or similar, it uses that
- Otherwise, it defaults to `"Disabled"` (battery-first selling)

This ensures compatibility across Deye inverter firmware versions.

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
