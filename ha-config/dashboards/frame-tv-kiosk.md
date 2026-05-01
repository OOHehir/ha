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

Current setup (post-2026-05-01): plain `sections`-layout dashboard rendered fullscreen by HAOSKiosk. The kiosk addon hides the toolbar and sidebar via its own config (`ha_sidebar: none`), so no `kiosk-mode` HACS install needed.

- Use the **sections** view type with `max_columns: 4` — predictable column layout, edit live in the UI editor
- Dark theme works well on the Frame TV QLED display
- Snapshot of the current dashboard config lives in `frame-tv-dashboard-config.json` and `frame-tv-kiosk-dashboard.yaml`
- For a graph card, install `apexcharts-card` via HACS — `data_generator` lets you plot dict attributes (e.g. Solcast hourly forecast)
- See "Lessons learned" below for what we tried and dropped

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

## Lessons learned (2026-05-01)

This section records dead ends and gotchas hit while iterating on this setup, so we don't re-walk them.

### Dropped wallpanel — kept HAOSKiosk

We initially used [lovelace-wallpanel](https://github.com/j-a-n/lovelace-wallpanel) for a Synology photo slideshow with cards as an info-box overlay. It's gone now. Friction that drove the decision:

- **Dual config trap.** Wallpanel uses its own `wallpanel.cards` list separate from `views[*].sections[*].cards`. The HA UI editor only writes to the view sections — so edits made on desktop *appeared to save* but never showed on the TV (because the TV was rendering wallpanel.cards). You end up keeping two copies of the same card definitions in sync by hand.
- **Hardcoded card width via CSS variable.** Each card is wrapped in a container whose width is `var(--wp-card-width)`, set inline by JS to `500px`. There is no `card_width` config option that takes effect — only overriding the variable in the style block does:
  ```yaml
  wallpanel:
    style:
      wallpanel-screensaver-info-box:
        --wp-card-width: "1800px"
  ```
- **Style block selector keys are element IDs without `#` prefix.** Valid IDs: `wallpanel-screensaver-info-box`, `wallpanel-screensaver-info-box-content`, `wallpanel-screensaver-overlay`, `wallpanel-screensaver-info-box-badges`, `wallpanel-screensaver-info-box-views`. The info-box itself defaults to `width: fit-content`; setting `width: 95vw` or `min-width` on it was overridden by wallpanel's inline style. The CSS variable approach was the only thing that actually worked.
- **`infoBoxContent` is a single-column CSS grid by default.** To get multi-column you have to either widen each card and let inner `grid`/`horizontal-stack` cards spread, or override `grid-template-columns` on `wallpanel-screensaver-info-box-content`.

A plain `sections` dashboard with `max_columns: 4` rendered fullscreen by HAOSKiosk gives 95% of the value with none of this. Trade-off: the rotating photo background is gone.

### Forecast.Solar (HA core integration) — no per-hour attributes

The HA Core `forecast_solar` integration (HACS-free) used to expose per-hour data as `wh_hours` / `watts` dict attributes on `sensor.energy_production_today`. Those were removed in 2024+; the integration now only exposes scalar sensors (today total, remaining today, peak time, this hour, next hour, etc.). The replacement action `forecast_solar.get_forecasts` is *not* registered on this install.

Result: you cannot draw a forward-looking hourly forecast bar chart from `forecast_solar` alone with apexcharts-card. We installed [Solcast PV Forecast](https://github.com/BJReplay/ha-solcast-solar) via HACS instead — it exposes the hourly forecast in `sensor.solcast_pv_forecast_forecast_today`'s `detailedHourly` attribute (list of `{period_start, pv_estimate, pv_estimate10, pv_estimate90}`), which apexcharts can plot directly via `data_generator`.

### Solcast — site ID is not the API key

The integration's "API key" field needs the **API key** from solcast.com → Account → API Toolkit (typically a 36-char UUID), NOT the **site ID** (also UUID-looking, lives in URLs like `/rooftop_sites/<site-id>/forecasts`). Pasting the site ID gets you a `404/Not Found` from the integration's `get sites` call (it's sent as a bearer token; no site matches).

Site IDs are auto-discovered from the API key — you don't enter them manually.

### Solcast — site capacity defaults to something useless

Solcast.com's site setup asks for "Total kW". If you skip it or leave a default, the forecast is generated against that capacity and clips at it. We saw `peak_forecast_today: 100000` (W) for a residential array because the site was registered with 100 kW capacity. Set this to your real array's kWp on solcast.com → My Sites → Edit before trusting any forecast values.

### apexcharts-card — duplicate registration breaks the card silently

HACS auto-registers Lovelace JS resources for installed cards (with a `?hacstag=...` query string). If you also manually add the resource via `Settings → Dashboards → Resources` (or `ha_config_set_dashboard_resource` over MCP), the card module loads twice. The second load throws `Failed to execute 'define' on 'CustomElementRegistry': the name "apexcharts-card-action-handler" has already been used` and the card silently fails to render.

Fix: keep only ONE resource entry. The HACS-managed one (with `?hacstag=...`) is preferred since HACS will keep it in sync with updates.

### HAOSKiosk + HA 2026.4+ auto-login

HAOSKiosk's built-in auto-login form-fill fails on HA 2026.4+ ("Auto-login failed: missing elements") due to shadow DOM changes in the login page. Fix is documented in Step 1 above (trusted_networks bypass).

## Hardware checklist

- [x] Micro-HDMI to HDMI cable connected
- [x] HDMI output confirmed
- [x] Cast integration working (`media_player.living_room_tv` — Google TV Streamer on HDMI 2)
- [x] HACS installed
- [x] HAOSKiosk add-on installed and configured (trusted networks auth required for auto-login)
- [x] Sonoff SNZB-06P paired via ZHA (`binary_sensor.sonoff_snzb_06p`)
- [x] apexcharts-card installed via HACS (for solar forecast graph)
- [x] Solcast PV Forecast installed via HACS and configured with API key + site capacity
- [x] Frame TV control via `samsungtv` (off) + `hdmi_cec` (on); Art Mode via `remote.send_command KEY_POWER`
- [x] Presence automation `automation.frame_tv_presence_on_off` (`automations/presence_frame_tv.yaml`)
- [x] Plain sections dashboard (no wallpanel, no card-mod, no kiosk-mode)
- [ ] Frame TV Art Mode / auto-source switching disabled in TV settings
- [ ] (Optional) Re-evaluate wallpanel/photo-slideshow if static dashboard feels too plain
