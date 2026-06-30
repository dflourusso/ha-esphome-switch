# DFLTech ESPHome Wall Switch

ESPHome firmware for a NodeMCU ESP8266 module that reads 6 GND inputs from a custom wall switch and sends click events to Home Assistant (single, double, and hold).

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

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose
- NodeMCU ESP8266 connected via USB (first flash only)
- Home Assistant with the [ESPHome integration](https://www.home-assistant.io/integrations/esphome/)

## Project layout

```
esphome-switch/
├── docker-compose.yml          # ESPHome dashboard + CLI container
├── dfltech-switch.yaml         # Device config (source of truth)
├── secrets.template.yaml       # Template for Wi-Fi / OTA credentials
└── container-config/           # Mounted as /config inside the container
    ├── dfltech-switch.yaml     # Copy or symlink from repo root
    └── secrets.yaml            # Your local credentials (not committed)
```

`container-config/` is gitignored. ESPHome reads configs from that directory at runtime.

## Setup

### 1. Prepare the config directory

```bash
mkdir -p container-config
cp dfltech-switch.yaml container-config/
cp secrets.template.yaml container-config/secrets.yaml
```

Edit `container-config/secrets.yaml` with your real values:

```yaml
wifi_ssid: "YourNetwork"
wifi_password: "YourPassword"
ap_password: "fallback-ap-password"
ota_password: "your-ota-password"
```

### 2. Start the ESPHome dashboard

```bash
docker compose up -d
```

Open the dashboard at **http://localhost:6052**.

### 3. Validate and compile

```bash
docker compose run --rm esphome config dfltech-switch.yaml
docker compose run --rm esphome compile dfltech-switch.yaml
```

Compiled firmware:

```
container-config/.esphome/build/dfltech-switch/.pioenvs/dfltech-switch/firmware.bin
```

## Flashing the device

### macOS (recommended workflow)

Docker on macOS cannot pass USB serial devices to containers reliably. Use this flow:

1. **Compile** in Docker (see above).
2. **First flash** via one of:
   - [ESPHome Web](https://web.esphome.io) — upload `firmware.bin` with the NodeMCU connected over USB.
   - **esptool** on the host:
     ```bash
     pip install esptool
     ls /dev/cu.*          # find your serial port, e.g. /dev/cu.usbserial-*
     esptool.py --port /dev/cu.usbserial-XXXX write_flash 0x0 \
       container-config/.esphome/build/dfltech-switch/.pioenvs/dfltech-switch/firmware.bin
     ```
3. **All later updates** over Wi-Fi via the dashboard (Install → Wirelessly) or:
   ```bash
   docker compose run --rm esphome run dfltech-switch.yaml
   ```

### Linux

USB passthrough works with Docker. Adjust the serial device in `docker-compose.yml` if needed (default: `/dev/ttyUSB0`), then:

```bash
docker compose run --rm esphome run dfltech-switch.yaml
```

## Home Assistant integration

The device connects via the native ESPHome API. Button actions are sent as events:

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

You can listen for events in **Developer Tools → Events** while testing.

## Common commands

| Task | Command |
|------|---------|
| Start dashboard | `docker compose up -d` |
| Stop dashboard | `docker compose down` |
| View logs | `docker compose logs -f esphome` |
| Validate config | `docker compose run --rm esphome config dfltech-switch.yaml` |
| Compile | `docker compose run --rm esphome compile dfltech-switch.yaml` |
| Flash / OTA | `docker compose run --rm esphome run dfltech-switch.yaml` |
| Clean build | `rm -rf container-config/.esphome/build/dfltech-switch` |

## Troubleshooting

**Config changes not picked up** — Make sure `container-config/dfltech-switch.yaml` matches the file you edited. Recompile after changes.

**Device won't join Wi-Fi** — Check `secrets.yaml`. If your router uses legacy WPA (not WPA2), add `min_auth_mode: WPA` under the `wifi:` block in `dfltech-switch.yaml`.

**No serial logs over USB** — Expected. `baud_rate: 0` frees GPIO3 (RX) for key 6. Use the ESPHome dashboard or Home Assistant for status instead.

**Stale build targeting wrong board** — Delete `container-config/.esphome/build/dfltech-switch` and recompile.

**Captive portal** — If Wi-Fi credentials are wrong, the device starts a fallback AP (`Dfltech-Switch Fallback`) so you can reconfigure via the dashboard.
