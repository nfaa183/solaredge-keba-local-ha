# SolarEdge PV + SolarEdge ONE EV Charger, fully local in Home Assistant

Solar-surplus EV charging with a **SolarEdge ONE EV Charger** (a rebranded KEBA KeContact P30
x-series) and a **SolarEdge inverter with DC-coupled battery**, without the SolarEdge cloud.

| Role | Component | Link |
| --- | --- | --- |
| Inverter, meter, battery (read) | SolarEdge Modbus, official HA integration, Modbus TCP | HA core |
| Charger measurements (read) | `keba_one_ev`, local UDP | [mousehome-net/keba_one_ev](https://github.com/mousehome-net/keba_one_ev) |
| Charger control (start/stop, current) | Local OCPP 1.6 server in HA | [lbbrhzn/ocpp](https://github.com/lbbrhzn/ocpp) |
| Charging logic | evcc, as a Home Assistant app, **no sponsor token** | [evcc.io](https://evcc.io) |
| Helper sensors | One HA package (this repo) | [`solaredge_keba.yaml`](homeassistant/packages/solaredge_keba.yaml) |
| Dashboard (optional) | Native cards + power-flow-card-plus | [`energy.yaml`](homeassistant/dashboards/energy.yaml) |

```
 SolarEdge ONE charger --OCPP 1.6 :9000--> [ OCPP server        ]
 (KEBA P30 x-series)   --UDP 7090-------> [ keba_one_ev        ]  Home Assistant  <--entities-->  evcc
 SE10K inverter        --Modbus TCP-----> [ SolarEdge Modbus   ]  + package                       (surplus logic)
 (meter + battery)
```

Result on the reference installation: the car charges on solar surplus with adjustable
thresholds, the home battery is never drained into the car, everything works without Internet.
Trade-off: the charger disappears from the SolarEdge app.

> Personal setup, shared as-is, tested in November 2026. Not affiliated with SolarEdge, KEBA,
> evcc or Home Assistant. Each step below ends with a check: do not go further until it passes.

## Reference installation

| Item | Version |
| --- | --- |
| Inverter | SolarEdge SE10K (SetApp), three-phase |
| Battery | 2 x SolarEdge 48 V, 4.6 kWh each (9.2 kWh), DC-coupled |
| Charger | SolarEdge ONE EV Charger = KEBA KC-P30 x-series, firmware 2.1.1, three-phase |
| Home Assistant | HAOS 2026.11.1 |
| keba_one_ev | v1.8.0 |
| OCPP integration | v0.12.3 |
| evcc | 0.319.0 (Home Assistant app) |
| power-flow-card-plus | v0.6.1 |

Since HA 2026.11, HACS is built in as **Settings -> Marketplace**. "Marketplace" below means that.

## Step 1. Inverter: Modbus TCP

1. Enable Modbus TCP on the inverter with SetApp (inverter Wi-Fi, then *Site Communication*).
2. In HA, add the **SolarEdge Modbus** integration with the inverter IP and the Modbus port.
3. Use **one** Modbus client only on the inverter (do not run a second Modbus integration).

**Check:** the inverter, meter and battery devices show live values: inverter DC power,
meter power, battery DC power, battery state of energy.

## Step 2. Charger measurements: keba_one_ev

1. Inside the charger, set **DIP switch 1.3 to ON** (enables the local UDP interface, port 7090).
2. Marketplace -> `keba_one_ev` -> install, then add the integration with the charger IP.

**Check:** with a car plugged in, *Vehicle Connected* is `on` and *Charging Power* follows the
car's charging power.

On the SolarEdge ONE firmware, only the current setpoint works over UDP: start/stop, energy
target and phase switching are ignored (the author removed them in v1.7.0 / v1.8.0). That is why
step 6 uses OCPP for control.

## Step 3. Rename the source entities

The package uses fixed entity ids. Default ids depend on device names and on the HA language
(`sensor.meter_1_power` in English, `sensor.meter_1_puissance` in French...), so rename these
entities once: open the entity, gear icon, field *Entity ID*.

| Source (integration, device, entity name) | New entity id | Example of default id |
| --- | --- | --- |
| SolarEdge Modbus, inverter, *DC power* | `sensor.solaredge_inverter_dc_power` | `sensor.solaredge_se10k_dc_power` |
| SolarEdge Modbus, battery, *DC power* | `sensor.solaredge_battery_dc_power` | `sensor.battery_1_dc_power` |
| SolarEdge Modbus, battery, *State of energy* | `sensor.solaredge_battery_soe` | `sensor.battery_1_state_of_energy` |
| SolarEdge Modbus, meter, *Power* | `sensor.solaredge_meter_power` | `sensor.meter_1_power` |
| keba_one_ev, *Charging* | `binary_sensor.keba_charging` | `binary_sensor.se_one_ev_charger_<ip>_charging` |
| keba_one_ev, *Vehicle Connected* | `binary_sensor.keba_vehicle_connected` | `binary_sensor.se_one_ev_charger_<ip>_vehicle_connected` |
| keba_one_ev, *Charging Power* | `sensor.keba_charging_power` | `sensor.se_one_ev_charger_<ip>_charging_power` |
| keba_one_ev, *Charging State* | `sensor.keba_charging_state` | `sensor.se_one_ev_charger_<ip>_charging_state` |
| keba_one_ev, *Session Energy* | `sensor.keba_session_energy` | `sensor.se_one_ev_charger_<ip>_session_energy` |
| keba_one_ev, *Total Energy* | `sensor.keba_total_energy` | `sensor.se_one_ev_charger_<ip>_total_energy` |
| keba_one_ev, *Current L1 / L2 / L3* | `sensor.keba_current_l1` / `_l2` / `_l3` | `sensor.se_one_ev_charger_<ip>_current_l1` |

Renaming keeps the history. Anything already using the old ids (dashboards, automations) must
be updated.

## Step 4. Helper sensors: the package

1. Copy [`homeassistant/packages/solaredge_keba.yaml`](homeassistant/packages/solaredge_keba.yaml)
   to `/config/packages/` (File editor or Studio Code Server app).
2. In `configuration.yaml`, add (or merge into an existing `homeassistant:` block):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. Developer tools -> YAML -> *Check configuration*, then restart Home Assistant.

It creates:

| Entity | Meaning |
| --- | --- |
| `sensor.pv_power` | PV production = inverter DC power + battery DC power, floor 0 |
| `sensor.grid_power` | Grid power, + import / - export |
| `sensor.battery_power` | Battery power, + discharge / - charge |
| `sensor.battery_charge_power`, `sensor.battery_discharge_power` | Battery power split in two |
| `sensor.house_power` | House consumption (car included) = PV + grid + battery |
| `sensor.charger_status_evcc` | `A` no car, `B` car connected, `C` charging |
| `sensor.pv_energy`, `sensor.battery_energy_charged`, `sensor.battery_energy_discharged` | kWh counters rebuilt from power |

**Check:**
- At night: `pv_power` = 0, and `house_power` = `grid_power` + `battery_power`.
- Car plugged in and not charging: `charger_status_evcc` = `B`.
- Every template of the package was rendered against the live entities of the reference
  installation and gave the same values as the sensors running there.

## Step 5. Energy dashboard

Settings -> Dashboards -> Energy:

| Field | Entity |
| --- | --- |
| Grid consumption / Return to grid | Meter *Energy imported* / *Energy exported* (SolarEdge Modbus) |
| Grid power | `sensor.grid_power` |
| Solar production | `sensor.pv_energy`, power `sensor.pv_power` |
| Battery: energy going in / out | `sensor.battery_energy_charged` / `sensor.battery_energy_discharged` |
| Battery power / state of charge | `sensor.battery_power` / `sensor.solaredge_battery_soe` |
| Individual device | `sensor.keba_total_energy`, power `sensor.keba_charging_power` |

**Check:** the next day, compare with the SolarEdge app (day view). On the reference
installation: production 14.0 / 14.00 kWh, grid import 58.5 / 58.82 kWh, export 167 / 168 Wh.

## Step 6. Charger control: local OCPP server

1. Marketplace -> **Open Charge Point Protocol (OCPP)** (lbbrhzn / drc38) -> install.
2. Add the integration: host `0.0.0.0`, port `9000`, secure connection off.
3. Charger web interface, OCPP page. **Write down the current SolarEdge values first** (rollback):

   | Field | Value |
   | --- | --- |
   | OCPP Communication Type | OCPP 1.6 JSON |
   | Central System Address | IP address of Home Assistant |
   | Central System Port | 9000 |
   | Central System Path | empty |
   | Secure Central System Connection | OFF |
   | Central System Authentication Method | BasicAuthentication, unchanged (keeps the SolarEdge key) |
   | Chargepoint Identity, Predefined Token | unchanged |

4. Restart the charger (allow several minutes). HA discovers it; configure it with:

   | Option | Value |
   | --- | --- |
   | Charge point identity | `charger` (entity prefix used by evcc and the dashboard) |
   | Maximum charging current | rating of your charger |
   | Charging sample interval | 10 |
   | Automatic detection of OCPP Measurands | on |
   | Force Smart Charging feature profile | **on** |
   | Send the station charging limit as an absolute schedule | **on** |

**Check:**
- Set `number.charger_maximum_current` to 10 A: *Current Offered* goes to 10 A within a minute.
  Without the "absolute schedule" option the charger answers `ChargePointMaxProfile: Rejected`.
- Start a new session (unplug / replug), then turn `switch.charger_charge_control` off: charging
  stops; back on: it restarts.

Pitfalls:
- *Charge Control* cannot stop a session that started before the charger connected to the
  server: the switch flips back to on. Unplug and replug once.
- Changing an option reloads the integration; the charger reconnects by itself after about 30 s.
- *Availability* may show off without blocking charging.
- With `ws://` the charger sends its BasicAuth credentials unencrypted on the LAN.

## Step 7. evcc

1. Settings -> Apps -> Install app -> menu -> Repositories: add
   `https://github.com/evcc-io/hassio-addon`, install **evcc** (not nightly), enable Watchdog
   and *Show in sidebar*, set `sqlite_file: /data/evcc.db`, start.
2. Configure the devices, either:
   - in the evcc web UI: manufacturer **Home Assistant**, instance `http://supervisor/core`
     (no login needed when evcc runs as an HA app), with the entities of
     [`evcc/evcc.yaml`](evcc/evcc.yaml); or
   - with that file (see its header).
3. Settings used on the reference installation (evcc web UI):

   | Setting | Value | Why |
   | --- | --- | --- |
   | Mode | Smart (solar) | Charge on surplus |
   | Phases | 3-phase | No 1P/3P switching without the SolarEdge cloud |
   | Min / max current | 6 A / 16 A | Max limited by the grid connection |
   | Solar share | 100 % in winter, 40 to 50 % mid-season | 100 % = start only if the surplus covers about 4.1 kW |
   | Battery: surplus priority | 0 % (car first) | The battery is topped up from the grid off-peak by a SolarEdge schedule in winter |
   | Battery: buffer for charging | 100 % (never) | The home battery never charges the car |

**Check:** on the evcc home page, grid, PV and battery power match the HA Energy dashboard,
and the charger shows the car as connected. With enough surplus in Smart mode, charging starts by
itself after about 1 minute and stops about 3 minutes after the surplus disappears.

## Step 8. Dashboard (optional)

1. Marketplace -> **power-flow-card-plus** -> install.
2. Settings -> Dashboards -> add dashboard -> open it -> edit -> menu -> *Raw configuration editor*
   -> paste [`homeassistant/dashboards/energy.yaml`](homeassistant/dashboards/energy.yaml).

## Known issues and limits

- **Battery energy counters over Modbus are not reliable.** On the reference installation they
  did not move for a whole day (0 kWh instead of about 2 kWh charge and 2 kWh discharge). The
  SolarEdge 48 V battery is known for counters that are not always increasing. The battery
  **power** is reliable, so the package rebuilds both counters from it.
- **Occasional wrong Modbus readings**: one battery power sample multiplied by 10 was seen.
  Negligible on the counters so far.
- **Three-phase only**: charging cannot go below 6 A x 3 x 230 V, about 4.1 kW.
- **evcc only reads the SolarEdge battery.** If a car charges while the battery discharges, the
  inverter feeds the car from the battery (for instance in *Fast* mode in the evening).
- **No vehicle identification**: the charger sends its predefined token, or the random tag
  generated by the OCPP integration for remote starts. Use the car maker's app for per-car stats.
- **The charger disappears from the SolarEdge app** and SolarEdge no longer pushes firmware updates.

## Rollback to SolarEdge

Charger OCPP page: address `csms.solaredge.com`, port 443, path `gw`, Secure ON,
BasicAuthentication, OCPP 1.6 JSON. Identity and key are still stored in the charger if you left
them untouched; it reconnects by itself.

## License

MIT
