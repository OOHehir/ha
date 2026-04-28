# Samsung Frame TV — HA Dashboard Kiosk via RPi5 HDMI

Display a Home Assistant dashboard on a Samsung Frame TV using the RPi5's HDMI output.

## Architecture

```
Sonoff SNZB-06P (Zigbee presence) --[ZHA]--> RPi5 (HAOS) --[micro-HDMI]--> Samsung Frame TV (HDMI input)
```

HDMI output is confirmed working (console visible). The remaining task is replacing the console with a fullscreen dashboard browser, and automating display on/off based on room presence.

## Step 1: Install HAOSKiosk add-on

HAOS has no desktop environment, so a kiosk add-on is needed to render a browser on the HDMI output. [HAOSKiosk](https://github.com/puterboy/HAOS-kiosk) runs an X server + Luakit browser inside a container and renders fullscreen to the physical HDMI output. It was developed specifically for RPi5 with HAOS.

1. In HA, go to **Settings > Add-ons > Add-on Store**
2. Open the three-dot menu and select **Repositories**
3. Add the HAOSKiosk repository:
   ```
   https://github.com/puterboy/HAOS-kiosk
   ```
4. Search for and install **HAOS Kiosk Display**
5. Configure the add-on with HA credentials and dashboard URL:
   ```
   URL: http://localhost:8123/lovelace/0
   ```
6. Enable **Start on boot** and **Watchdog**
7. Start the add-on — the HDMI output should switch from console to the dashboard

### Auto-login fix: Trusted Networks

HAOSKiosk's built-in auto-login fails on HA 2026.4+ ("Auto-login failed: missing elements") due to shadow DOM changes in the login page. The fix is to bypass the login form entirely using trusted networks auth. Add this to `configuration.yaml`:

```yaml
homeassistant:
  auth_providers:
    - type: trusted_networks
      trusted_networks:
        - 127.0.0.1/32
        - ::1/128
      allow_bypass_login: true
    - type: homeassistant
```

Both `127.0.0.1/32` and `::1/128` (IPv6 localhost) are required — the kiosk add-on connects via IPv6 loopback. Restart HA after adding this.

> **TODO:** `allow_bypass_login: true` skips authentication for all localhost connections, not just the kiosk. This means any process on the RPi can access HA without credentials. Investigate restricting this to the kiosk add-on only, or use a dedicated kiosk user with limited permissions.

> **Note:** HAOSKiosk uses Luakit (webkit-based), not Chromium. Most dashboard cards render fine, but very complex custom cards may behave slightly differently.

## Step 2: Samsung Frame TV — Art Mode considerations

The Frame TV's Art Mode activates when it detects idle state. To prevent it switching away from the HDMI input:

- Disable **Motion Sensor** in TV Settings > General > Art Mode Settings
- Set the TV input to the HDMI port and disable auto-source switching
- Consider using a CEC command from HA to keep the TV on the correct input

## Step 3: Dashboard design tips for a wall display

- Use a **Panel (single card)** dashboard or a dedicated view with `panel: true`
- Dark theme works well on the Frame TV QLED display
- Hide the sidebar and header for a clean kiosk look:
  - Install **kiosk-mode** via HACS > Frontend
  - This hides the sidebar, header, and overflow menu
- Suggested cards: weather, time, energy (Zappi), room status
- Use `browser_mod` integration for periodic refresh if needed

## Step 4: Presence-based display control with Sonoff SNZB-06P

Use a **Sonoff SNZB-06P** (Zigbee mmWave presence sensor) to turn the Frame TV on when someone is in the room and off when the room is empty.

### Sensor setup

1. Pair the SNZB-06P via **ZHA** (you have a ZBT-2 Zigbee coordinator)
   - Put ZHA in pairing mode: **Settings > Devices > Add Device (ZHA)**
   - Hold the SNZB-06P button for 5s to enter pairing mode
2. Once paired, it exposes `binary_sensor.sonoff_snzb_06p` (confirmed entity ID)
3. Mount the sensor with line of sight to the seating area — mmWave detects stationary presence, not just motion

### TV control via Cast integration

The Frame TV is already available in HA as `media_player.living_room_tv` via the Cast integration. This provides power control directly — no CEC or shell commands needed.

**Test via Developer Tools > Services:**

```yaml
# Turn TV on
action: media_player.turn_on
target:
  entity_id: media_player.living_room_tv

# Turn TV off
action: media_player.turn_off
target:
  entity_id: media_player.living_room_tv
```

### Automation: turn TV on/off based on presence

**Turn on when occupied:**
```yaml
alias: Frame TV — turn on when room occupied
trigger:
  - platform: state
    entity_id: binary_sensor.sonoff_snzb_06p
    to: "on"
action:
  - action: media_player.turn_on
    target:
      entity_id: media_player.living_room_tv
```

**Turn off when empty (with delay):**
```yaml
alias: Frame TV — turn off when room empty
trigger:
  - platform: state
    entity_id: binary_sensor.sonoff_snzb_06p
    to: "off"
    for: "00:05:00"  # 5 min delay to avoid flickering
action:
  - action: media_player.turn_off
    target:
      entity_id: media_player.living_room_tv
```

### Notes

- The SNZB-06P uses 5.8GHz mmWave so it detects stationary people (sitting on a couch) — much better than PIR for this use case
- The 5-minute `for:` delay on the off automation prevents the TV flickering if the sensor briefly loses detection
- Cast integration handles TV power over the network — the existing `media_player.living_room_tv` entity is confirmed working
- If Cast power control proves unreliable, HDMI-CEC is a fallback — add `hdmi_cec:` to `configuration.yaml` and the RPi5 can control the TV over the HDMI cable directly
- Some Samsung TVs need CEC enabled in settings: **Settings > General > External Device Manager > Anynet+ (HDMI-CEC) > On**

## Hardware checklist

- [x] Micro-HDMI to HDMI cable connected
- [x] HDMI output confirmed (console visible)
- [x] Cast integration working (`media_player.living_room_tv`)
- [x] HACS installed
- [x] HAOSKiosk add-on installed and configured (trusted networks auth required for auto-login)
- [x] Sonoff SNZB-06P paired via ZHA (`binary_sensor.sonoff_snzb_06p`)
- [x] Dashboard designed for wall display (wallpanel photo slideshow + card-mod transparent cards)
- [x] Synology Photos mounted via SMB (`/media/synology_photos` from `ds.local`)
- [x] wallpanel installed via HACS
- [ ] card-mod installed via HACS (needed for transparent card styling)
- [ ] Sonoff SNZB-06P mounted in room with line of sight to seating area
- [ ] Frame TV Art Mode / auto-source switching disabled
- [ ] Presence automation created and tested
- [ ] kiosk-mode installed via HACS
