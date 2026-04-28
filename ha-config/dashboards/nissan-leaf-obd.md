# Nissan Leaf — OBD2 via LELink2 BLE

Read Nissan Leaf battery, range, and diagnostic data in Home Assistant using a LELink2 Bluetooth Low Energy OBD2 adapter plugged into the car's OBD port.

## Architecture

```
LELink2 (BLE OBD2 in Leaf) --[BLE]--> RPi5 (HAOS) --[HA integration]--> entities
```

The LELink2 is a BLE-only adapter (no Bluetooth Classic). It advertises as `OBDBLE` (MAC: `C4:BE:84:F3:E7:BE`). It has zero-power standby and wakes when the car's 12V system is active (open the driver's door or turn to ACC). The RPi5 running HAOS needs to be within BLE range of the car (~10m, depending on walls/garage layout).

> **Note:** The RPi5's onboard WiFi and Bluetooth share the same radio (Infineon CYW43455). Running BLE scans can cause WiFi drops. Mitigations, in order of effectiveness:
> 1. Put HA on ethernet (or a powerline adapter) so WiFi drops don't matter.
> 2. Use an **ESP32 Bluetooth Proxy** placed near the car (see below) — moves BLE off the Pi entirely and extends range.
> 3. Add a USB BT adapter (e.g. **TP-Link UB500**, BT 5.0, on HA's known-working list). Avoid the older UB400 (CSR8510, BT 4.0) — works but not recommended.

## Option A: Direct BLE — nissan_leaf_obd_ble custom component

This is the most direct approach — HA connects to the LELink2 over BLE with no phone or intermediary app required.

- GitHub: [pbutterworth/nissan-leaf-obd-ble](https://github.com/pbutterworth/nissan-leaf-obd-ble)
- Community thread: [Custom component Nissan Leaf via LeLink 2 (ELM327) BLE](https://community.home-assistant.io/t/custom-component-nissan-leaf-via-lelink-2-elm327-ble/561961)

### Prerequisites

- HA Bluetooth integration enabled (confirmed active on RPi5)
- LELink2 plugged into the Leaf's OBD port (under the steering column, left side)
- Car 12V must be awake (open driver's door) so the LELink2 advertises as `OBDBLE`

### Install

1. Add custom repository in HACS:
   - **HACS > Integrations > three-dot menu > Custom repositories**
   - Repository: `https://github.com/pbutterworth/nissan-leaf-obd-ble`
   - Category: **Integration**
2. Search HACS for **Nissan Leaf OBD BLE** and install (installed v0.3.1b2)
3. Restart HA
4. **Settings > Devices & Services > Add Integration > Nissan Leaf OBD BLE**
   - Select `OBDBLE (C4:BE:84:F3:E7:BE)` from the discovered devices
   - If "no unconfigured devices" appears, the car's 12V is asleep — open the driver's door and retry

### Entities

Once connected, the integration creates 34 entities:

**Battery & Charging:**
- `sensor.nissan_leaf_obd_ble_state_of_charge` — SOC (%)
- `sensor.nissan_leaf_obd_ble_hv_battery_health` — SOH (%)
- `sensor.nissan_leaf_obd_ble_hv_battery_capacity` — capacity (Ah)
- `sensor.nissan_leaf_obd_ble_hv_battery_voltage` — HV pack voltage (V)
- `sensor.nissan_leaf_obd_ble_hv_battery_current_1` / `_2` — current (A)
- `sensor.nissan_leaf_obd_ble_range_remaining` — range (km)
- `sensor.nissan_leaf_obd_ble_charging_mode` — charging mode
- `sensor.nissan_leaf_obd_ble_on_board_charger_output_power` — charger power (W)
- `sensor.nissan_leaf_obd_ble_number_of_l1_l2_charges` / `_quick_charges` — charge counts

**Driving:**
- `sensor.nissan_leaf_obd_ble_vehicle_speed` — speed (km/h)
- `sensor.nissan_leaf_obd_ble_odometer` — odometer (km)
- `sensor.nissan_leaf_obd_ble_motor_rpm` — motor RPM
- `sensor.nissan_leaf_obd_ble_gear_position` — gear
- `sensor.nissan_leaf_obd_ble_traction_motor_power` — motor power (W)

**Tyres:**
- `sensor.nissan_leaf_obd_ble_tyre_pressure_front_left` / `_front_right` / `_rear_left` / `_rear_right` — pressures (kPa)

**Climate:**
- `sensor.nissan_leaf_obd_ble_ambient_temperature` — ambient temp (°C)
- `sensor.nissan_leaf_obd_ble_ac_system_power` — AC power (W)
- `binary_sensor.nissan_leaf_obd_ble_ac_status` / `_rear_heater_status`

**12V System:**
- `sensor.nissan_leaf_obd_ble_12v_battery_voltage` — 12V voltage (V)
- `sensor.nissan_leaf_obd_ble_12v_battery_current` — 12V current (A)

**Modes & Controls:**
- `binary_sensor.nissan_leaf_obd_ble_eco_mode_status` / `_e_pedal_mode_status` / `_power_switch_status`
- `button.nissan_leaf_obd_ble_refresh` — manual refresh

### Limitations

- BLE range: the RPi5 must be close enough to reach the car. If the Leaf is parked in a garage adjacent to the RPi5, this usually works. Outdoor parking further away may not.
- The LELink2 draws a small amount from the 12V battery. For cars parked for extended periods, consider unplugging it.
- Polling frequency: the component polls periodically. Frequent polling can drain the 12V battery faster.

### Improving range and avoiding WiFi/BT coexistence: ESP32 Bluetooth Proxy

If the RPi5 is too far from the car, or BLE scans are interfering with WiFi, run an [ESPHome Bluetooth Proxy](https://esphome.io/projects/?type=bluetooth-proxy) on a spare ESP32 placed near the parking spot. HA discovers the LELink2 through the proxy with no integration changes.

```
LELink2 (in Leaf) --[BLE]--> ESP32 proxy (garage/window) --[WiFi]--> RPi5 (HAOS)
```

**Board choice (best → workable):**

- **ESP32-S3** — best. BLE 5.0 with Coded PHY (long range), ample RAM, supports active connections (required for `nissan_leaf_obd_ble` to *talk* to the LELink2, not just observe).
- **ESP32-C6 / C3** — also fine. BLE 5.0, active connections supported (C3 has fewer concurrent slots).
- **Original ESP32 (WROOM/WROVER)** — works for a single OBD adapter at modest range. BLE 4.2.
- **ESP32-S2** — no Bluetooth radio. ❌
- **ESP32-H2** — no WiFi. ❌

**Flash (no code to write):**

1. Plug ESP32 into laptop via USB.
2. Open https://esphome.github.io/bluetooth-proxies/ in Chrome/Edge (WebSerial).
3. Select **Bluetooth Proxy** → variant matching your board (e.g. *Generic ESP32-S3*).
4. **Connect** → pick the serial port → **Install**.
5. After flashing, join the captive WiFi AP it broadcasts and supply your home WiFi creds.
6. In HA: **Settings → Devices & Services** — ESPHome auto-discovers the proxy. **Configure** to add.
7. Place the ESP32 on USB power near the car (garage outlet, kitchen window).

The LELink2 should reappear under the existing `nissan_leaf_obd_ble` integration via the proxy. If not: **Settings → Devices & Services → Bluetooth → Configure** and confirm the proxy is listed as an adapter with active scanning enabled.

Source/YAML for the firmware: https://github.com/esphome/bluetooth-proxies

## Option B: LeafSpy Pro + ha-leafspy

Use the **LeafSpy Pro** Android/iOS app as a bridge between the LELink2 and HA.

### How it works

```
LELink2 --[BLE]--> Phone (LeafSpy Pro) --[HTTP POST]--> HA (ha-leafspy)
```

### Install

1. Install **LeafSpy Pro** on your phone and pair it with the LELink2
2. Install **ha-leafspy** in HA:
   - HACS > Integrations > search **LeafSpy** > Install
   - Or manually: [github.com/jesserockz/ha-leafspy](https://github.com/jesserockz/ha-leafspy)
   - Restart HA
3. Add the integration: **Settings > Devices & Services > Add Integration > LeafSpy**
4. In LeafSpy Pro, configure the server to send data to your HA instance:
   - **Settings > Server** in LeafSpy
   - Set URL to your HA address (e.g., `http://<ha-ip>:8123`)
5. LeafSpy sends periodic updates whenever the app is open and connected to the LELink2

### Trade-offs vs Option A

- **Pro:** LeafSpy decodes more Leaf-specific data (cell voltages, trip stats, charging history)
- **Pro:** Well-tested with LELink2 specifically
- **Con:** Requires your phone to be in the car with the app running
- **Con:** Not automatic — depends on you opening the app

## Option C: Torque Pro + MQTT (generic OBD2)

A more generic approach using **Torque Pro** (Android only) to publish OBD2 data to HA via MQTT.

### How it works

```
LELink2 --[BLE]--> Phone (Torque Pro) --[MQTT]--> HA
```

This works for any OBD2 vehicle, not just the Leaf. However, it doesn't decode Leaf-specific battery data (SOH, GIDs, cell voltages) — only standard OBD2 PIDs.

See: [Torque2MQTT](https://community.home-assistant.io/t/torque2mqtt-get-data-about-your-automobile-into-mqtt-home-assistant/157376) or the built-in [HA Torque integration](https://www.home-assistant.io/integrations/torque/).

## Dashboard

SOC and key Leaf sensors are displayed on the **Frame TV** dashboard in the "Nissan Leaf" card (column 2, alongside Energy and EV Charger). SOC is also shown in the wallpanel screensaver EV Charger card.

Entities shown on dashboard:
- Battery (SOC %)
- Range remaining (km)
- Battery health (SOH %)
- 12V battery voltage (V)

## Recommendation

For a Nissan Leaf + LELink2 on HAOS with an RPi5:

- **Try Option A first** (direct BLE) — no phone dependency, fully automated
- **Fall back to Option B** (LeafSpy) if BLE range is a problem or you want richer Leaf-specific data on demand

## Setup checklist

- [x] LELink2 plugged into Leaf OBD port
- [x] Car area defined in HA (`area_id: car`)
- [x] Zappi EV charger integrated (`zappi-17004798`)
- [x] HA Bluetooth integration confirmed active
- [x] LELink2 discovered in HA Bluetooth scan (`OBDBLE` / `C4:BE:84:F3:E7:BE`)
- [x] nissan_leaf_obd_ble v0.3.1b2 installed via HACS (Option A)
- [x] 34 Leaf entities created in HA
- [x] Device assigned to Car area
- [x] SOC + key sensors added to Frame TV dashboard
- [ ] Confirm entities populate with data when car is awake
- [ ] Verify BLE range from RPi5 to parking spot is reliable
- [ ] If range/coexistence is a problem: flash a spare ESP32 as a Bluetooth Proxy and place near the car
