# DFLTech Switch

ESPHome firmware for an ESP32-C3 Super Mini wall switch with 6 inputs (single, double, and hold events sent to Home Assistant).

## First-time installation

Connect the Super Mini via USB-C, then use the button below to flash the factory firmware from your browser (Chrome or Edge required).

If the flasher cannot connect, hold **BOOT**, tap **RST**, then release **BOOT**, and try again.

<script type="module" src="https://unpkg.com/esp-web-tools@10/dist/web/install-button.js?module"></script>

<esp-web-install-button manifest="firmware/manifest.json">
  <button slot="activate">Install DFLTech Switch</button>
  <span slot="unsupported">Your browser does not support WebSerial. Use Chrome or Edge on desktop.</span>
  <span slot="not-allowed">HTTPS is required (or use localhost).</span>
</esp-web-install-button>

## After flashing

1. The device starts a Wi-Fi access point named `dfltech-switch-XXXXXX` (password: `dfltech-setup`).
2. Connect with your phone — the captive portal opens automatically (or go to http://192.168.4.1/).
3. Enter your home Wi-Fi credentials.
4. Add the device in Home Assistant via the ESPHome integration (`dfltech-switch-XXXXXX.local`).

Firmware updates are offered automatically in Home Assistant when a new release is published.

## Factory reset

Hold the **BOOT** button (GPIO9) on the Super Mini for **10 seconds**, then release. Wi-Fi credentials are erased and the setup access point starts again.
