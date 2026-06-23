> **See the main [README.md](../README.md) at the repo root for full documentation.**

# Tibber Deye Smart Arbitrage v9 - Boundary and Force-Sell Fix

This version fixes the issue visible in the Solarman screenshots where Time 1 and Time 2 both started at `00:00`.
On Deye/Solarman TOU screens, the next program start time becomes the previous program end time, so duplicate start times can create a zero-length period such as `00:00 -> 00:00`.

## Main fixes in v9

- Keeps the capitalized Solarman select options: `Grid` and `Disabled`.
- Avoids a duplicate midnight TOU boundary when the cheapest charge block starts at `00:00`.
- If charging runs from `00:00` to something like `04:00`, the period after charging resumes night self-use at about `600 W` until the morning grid-charge window.
- Adds `input_boolean.tibber_deye_force_sell_export`, default `on`, so the high-price sell/export block is written even if the spread guard would otherwise block it.
- Keeps normal grid charging at `5000 W`.
- Keeps sell/export at `10000 W`.
- Keeps sell stop SOC at `25%` and night stop SOC at `20%`.
- Adds `input_number.tibber_deye_very_expensive_price_eur_kwh` (default `0.35`) to prioritize sell slots considered very expensive.
- Blocks automatic selling when selected sell average price is negative (unless force sell/export is enabled).
- Adds a companion grid-charging-current schedule for `number.inverter_battery_grid_charging_current`: `20 A` at 08:00, `40 A` at 09:00, `50 A` at 10:00, and `110 A` during the cheapest Tibber slot.

## Install - recommended package method

Copy:

```text
tibber_deye_smart_arbitrage_package_v9_boundary_force_sell.yaml
```

to:

```text
/config/packages/tibber_deye_smart_arbitrage_package.yaml
```

Make sure `configuration.yaml` contains:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then run:

```text
Developer Tools -> YAML -> Check configuration
Restart Home Assistant
```

Run the automation:

```text
Tibber Deye - Smart Dynamic Charge and Sell Schedule v9
```


The active package should use `Grid` and `Disabled`, not lowercase options.

## Expected behaviour for the schedule in your screenshot

If the selected cheapest block is `00:00 -> 04:00`, v9 should write something like:

```text
00:00 -> 04:00  Grid charge, 5000 W, Batt 90%
04:00 -> 08:00  Night self-use, 600 W, Batt 20%
08:00 -> charge  Grid charge enabled, current ramp starts at 20 A
09:00            Grid charge current rises to 40 A
10:00            Grid charge current rises to 50 A
cheapest slot    Grid charge current rises to 110 A
sell block       Sell/export, 10000 W, Batt 25%
after sell       House self-use, 600 W, Batt 20%
```

It should no longer produce a useful slot with `00:00 -> 00:00`.

## Tuning helpers

- `input_number.tibber_deye_charge_power_w`: default `5000`
- `input_number.tibber_deye_sell_power_w`: default `10000`
- `input_number.tibber_deye_night_power_w`: default `600`
- `input_number.tibber_deye_very_expensive_price_eur_kwh`: default `0.35`
- `input_number.tibber_deye_grid_charge_current_08_a`: default `20`
- `input_number.tibber_deye_grid_charge_current_09_a`: default `40`
- `input_number.tibber_deye_grid_charge_current_10_a`: default `50`
- `input_number.tibber_deye_grid_charge_current_cheapest_a`: default `110`
- `input_number.tibber_deye_charge_target_soc`: default `90`
- `input_number.tibber_deye_sell_stop_soc`: default `25`
- `input_number.tibber_deye_night_stop_soc`: default `20`
- `input_boolean.tibber_deye_force_sell_export`: default `on`
- `input_boolean.tibber_deye_debug_notifications`: default `off`

Turn on debug notifications before running the automation if you want the chosen charge/sell windows shown as a Home Assistant persistent notification.
