# DFLTech ESPHome Wall Switch

ESPHome firmware for a NodeMCU ESP8266 module that reads 6 GND inputs from a custom wall switch and sends click events to Home Assistant (single, double, and hold).

Distributed as a **product firmware**: flash once, provision Wi-Fi via captive portal, receive OTA updates through Home Assistant.

## Hardware

| Key | NodeMCU pin | GPIO | Notes |
|-----|-------------|------|-------|
| 1 | D1 | GPIO5 | |
| 2 | D2 | GPIO4 | |
| 3 | D5 | GPIO14 | |
| 4 | D6 | GPIO12 | |
| 5 | D7 | GPIO13 | |
| 6 | RX | GPIO3 | Uses internal pull-up; serial logging is disabled |

Board: **NodeMCU v2** (`nodemcuv2`). Each input is active-low (switch connects to GND). Internal pull-ups are enabled in firmware.

## End users

### Install firmware (first time)

1. Download the latest `*.factory.bin` from [GitHub Releases](https://github.com/dflourusso/ha-esphome-switch/releases), **or** use the browser installer at [dflourusso.github.io/ha-esphome-switch](https://dflourusso.github.io/ha-esphome-switch/) (Chrome/Edge, USB connected).
2. After flashing, the device creates a Wi-Fi access point named `dfltech-switch-XXXXXX` (password: `dfltech-setup`).
3. Connect with your phone — the captive portal opens (or open http://192.168.4.1/).
4. Enter your home Wi-Fi credentials.
5. In Home Assistant, add the device via **Settings → Devices & services → ESPHome** (`dfltech-switch-XXXXXX.local`).

### OTA updates

When a new version is published, Home Assistant shows a **Firmware** update on the device. Install from the device page or **Settings → Updates**.

### Factory reset

Hold the **FLASH** button (GPIO0) on the NodeMCU for **10 seconds**, then release. Wi-Fi credentials are cleared and the setup access point starts again.

### Device naming

Each device gets a unique hostname (`dfltech-switch-aabbcc`) from its MAC address. After adding to Home Assistant, rename the device in the ESPHome integration UI if you want a friendlier label (e.g. "Kitchen Switch").

## Developers

### Project layout

```
esphome-switch/
├── dfltech-switch.yaml          # Core device logic (keys, captive portal, factory reset)
├── dfltech-switch.factory.yaml  # Distribution build (HTTP OTA + update entity)
├── dfltech-switch.dev.yaml      # Local dev overlay (Wi-Fi from secrets)
├── secrets.template.yaml        # Template for local secrets.yaml
├── static/                      # GitHub Pages installer site
└── .github/workflows/           # CI, release, and Pages deploy
```

### Local compile

**Factory image (what you ship):**

```bash
docker run --rm -v "$PWD:/config" ghcr.io/esphome/esphome compile dfltech-switch.factory.yaml
```

Output: `.esphome/build/dfltech-switch/.pioenvs/dfltech-switch/firmware.bin` (or use the `*.factory.bin` from CI).

**Dev build (your Wi-Fi credentials):**

```bash
cp secrets.template.yaml secrets.yaml
# edit secrets.yaml
docker run --rm -v "$PWD:/config" ghcr.io/esphome/esphome compile dfltech-switch.dev.yaml
```

Or with ESPHome installed locally: `esphome compile dfltech-switch.factory.yaml`

### Publish a release

1. Go to **Actions → Release Firmware → Run workflow**.
2. Enter a version (e.g. `1.0.0`) and optional release notes.
3. The workflow builds firmware, creates a GitHub Release with `.factory.bin` / `.ota.bin`, and deploys GitHub Pages with the OTA manifest.

OTA manifest URL (devices poll this): `https://dflourusso.github.io/ha-esphome-switch/firmware/manifest.json`

### Flashing on macOS

Docker cannot pass USB serial reliably on macOS. Compile in Docker, then flash via:

- [ESPHome Web](https://web.esphome.io) — upload `firmware.bin` over USB
- **esptool** on the host:
  ```bash
  pip install esptool
  esptool.py --port /dev/cu.usbserial-XXXX write_flash 0x0 firmware.bin
  ```

## Home Assistant integration

Button actions are sent as events:

| Event type | `esphome.dfltech_switch` |
|------------|--------------------------|
| `key` | `"1"` … `"6"` |
| `action` | `single`, `double`, `hold` |

Hold requires the button to be pressed for at least **1.2 seconds**.

Example automation:

```yaml
automation:
  - alias: "Wall switch key 1 single"
    trigger:
      - platform: event
        event_type: esphome.dfltech_switch
        event_data:
          key: "1"
          action: single
    action:
      - service: light.toggle
        target:
          entity_id: light.living_room
```

## Troubleshooting

**Device won't join Wi-Fi** — Use factory reset (FLASH 10s), then re-provision via captive portal.

**No serial logs over USB** — Expected. `baud_rate: 0` frees GPIO3 (RX) for key 6. Use Home Assistant for status.

**OTA check fails** — Ensure GitHub Pages is deployed after a release. ESP8266 TLS may need buffer tuning in `dfltech-switch.factory.yaml` (`tls_buffer_size_rx`).
