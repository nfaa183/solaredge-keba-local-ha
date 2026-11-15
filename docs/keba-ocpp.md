# Local OCPP server for the SolarEdge ONE / KEBA P30 charger

The charger talks OCPP 1.6 to a single server. Pointing it at Home Assistant instead of SolarEdge
gives start/stop and current control, which UDP does not provide on this firmware.

## 1. OCPP server in Home Assistant

1. Install **Open Charge Point Protocol (OCPP)** by lbbrhzn / drc38
   (Settings -> Marketplace, formerly HACS).
2. Add the integration: host `0.0.0.0`, port `9000`, secure connection **off**.

## 2. Charger OCPP page (charger web interface)

| Field | Value |
| --- | --- |
| OCPP Communication Type | OCPP 1.6 JSON |
| Central System Address | IP address of Home Assistant |
| Central System Port | 9000 |
| Central System Path | empty |
| Secure Central System Connection | OFF |
| Central System Authentication Method | BasicAuthentication, left as is |
| Chargepoint Identity, Predefined Token | unchanged |

Before changing anything, note the original SolarEdge values (address, port, path) for rollback.
Leaving BasicAuthentication in place avoids wiping the key provided by SolarEdge; the Home
Assistant server accepts the connection anyway. After the charger restarts (several minutes),
it shows up in Home Assistant.

Note: with `ws://` the charger sends its BasicAuth credentials unencrypted on your LAN. Keep the
charger and Home Assistant on networks you trust.

## 3. Options of the discovered charger

| Option | Value |
| --- | --- |
| Charge point identity | `charger` (entity prefix) |
| Maximum charging current | rating of your charger |
| Charging sample interval | 10 s |
| Force Smart Charging feature profile | checked |
| Send the station charging limit as an absolute schedule | **checked** |

Without the absolute schedule option the charger rejects the current limit
(`ChargePointMaxProfile: Rejected`). With it, **Maximum Current** is applied
(check: *Current Offered* follows the setpoint).

## Pitfalls

- **Charge Control** only stops sessions that started after the charger connected to the server.
  For an older session the switch flips back to ON by itself: unplug and replug the car once.
- Changing an option reloads the integration; the charger reconnects by itself after about 30 s.
- The **Availability** switch may stay OFF without preventing charging.
- The id tag is not a vehicle id: it is the charger's predefined token, or the random tag the
  integration generates for remote starts.

## Entities used by evcc

- `switch.charger_charge_control` : start / stop
- `number.charger_maximum_current` : current setpoint
