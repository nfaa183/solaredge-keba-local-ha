# evcc on top of Home Assistant (no sponsor token)

evcc decides when to charge and at what current. Every device uses evcc's **Home Assistant**
template, so no sponsor token is needed (evcc's native OCPP and KEBA Modbus chargers require one).

## Install

1. Home Assistant -> Settings -> Apps -> add repository `https://github.com/evcc-io/hassio-addon`.
2. Install **evcc** (not nightly), enable Watchdog and "Show in sidebar".
3. Configuration tab: `sqlite_file: /data/evcc.db`. Start, set the admin password, then configure
   everything from the evcc web UI.

## Devices (template: Home Assistant)

Entity ids match `homeassistant/templates.yaml`; adapt the keba_one_ev ids to yours.

| Device | Field | Entity |
| --- | --- | --- |
| Grid meter | power | `sensor.grid_power` (positive = import) |
| PV meter | power | `sensor.pv_power` |
| Battery | power | `sensor.battery_power` (positive = discharge) |
| Battery | soc | `sensor.battery_1_state_of_energy` |
| Charger | status | `sensor.charger_status_evcc` |
| Charger | enabled + enable | `switch.charger_charge_control` (OCPP) |
| Charger | setMaxCurrent | `number.charger_maximum_current` (OCPP) |
| Charger | power | `sensor.keba_charging_power` |
| Charger | energy | `sensor.keba_total_energy` |
| Charger | currents L1/L2/L3 | `sensor.keba_current_l1` / `_l2` / `_l3` |
| Charger | phase switch | leave empty |

## Settings I use

| evcc setting | Value | Why |
| --- | --- | --- |
| Mode | Smart (solar) | Charge on solar surplus |
| Phases | 3-phase | No local 1P/3P switching |
| Min / max current | 6 A / 20 A | Max kept below the 32 A grid connection (heat pump); "Fast" mode for a full-power charge |
| Solar share | 100 % in winter, 50 % mid-season | 100 % = start only when the surplus covers about 4.1 kW |
| Battery boost | Off | |
| Cheap grid charging | Off | Depends on your tariff windows |
| Surplus priority (Battery page) | 0 %, car first | In winter the battery is topped up from the grid during off-peak hours by a SolarEdge schedule |
| Battery as charging buffer | 100 % (never) | The home battery never charges the car |

These values depend on your tariff, your grid connection and your habits: treat them as an example.
