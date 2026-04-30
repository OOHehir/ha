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

> **Gotcha — display must be connected at addon start (2026-04-30):** HAOSKiosk checks DRM connector state on startup and exits with `ERROR: No connected video card detected. Exiting..` if both HDMI ports report `disconnected`. The watchdog gives up after a few retries and the addon stays in `error` state — it does **not** auto-recover when the display comes back. Symptom: TV shows the Linux console (RPi5's bare framebuffer) instead of the dashboard. Fix: ensure the Frame is on HDMI 1 (RPi5) and `hassio.addon_start` the addon. Long-term: an automation that restarts the addon on `presence_on` would harden this.

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

### TV control

> **Note (2026-04-30):** `media_player.living_room_tv` is the Google TV Streamer, not the Frame. Frame control is split across two integrations:
>
> | Action | Service | Why |
> |---|---|---|
> | **ON** | `hdmi_cec.power_on` | Broadcasts Active Source so the Frame wakes AND switches to **HDMI 1** (the RPi5). `samsungtv`'s `media_player.turn_on` cannot wake the Frame from cold (no WoL/MAC) and would not switch input. |
> | **OFF** | `media_player.turn_off` on `media_player.samsung_the_frame_qe32ls03tbkxxu` | Directed CEC `<Standby>` is silently ignored by Frame TVs (verified by manual test). The built-in `samsungtv` integration's `turn_off` works reliably. |
> | **Art Mode** | `remote.send_command` with `KEY_POWER` on `remote.samsung_the_frame_qe32ls03tbkxxu` | TV must be on first. KEY_POWER toggles between full-on and Art Mode. There's no dedicated art-mode service in the built-in integration. |
>
> Note: `media_player.select_source` on the Frame only offers `["TV", "HDMI"]` and "HDMI" routes to **HDMI 2** (the Streamer), not HDMI 1.

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
