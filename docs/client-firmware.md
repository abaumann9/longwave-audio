# Client Firmware

This guide covers flashing the snapclient firmware to a XIAO ESP32-S3 Plus endpoint and provisioning it onto your Wi-Fi HaLow network.

There are two flashing paths:

1. **[Web Flasher](#flash-via-web-preferred)** — recommended for most users. Browser-based, no toolchain required.
2. **[Build from Source](#build-from-source-developers)** — for firmware developers and contributors.

After flashing, endpoints can be provisioned via the **[ESP SoftAP Provisioning](#provisioning-esp-softap)** mobile app.

---

## Flash via Web (Preferred)

Prebuilt firmware is published to GitHub Pages and flashed directly from the browser using [ESP Web Tools](https://esphome.github.io/esp-web-tools/). No build environment required.

### Requirements

- Chrome or Edge on desktop (Web Serial API)
- USB-C cable
- An assembled endpoint (see [Hardware Guide](hardware-guide.md))
   - Temporarily remove the Audio DAC header for flashing

### Steps

1. Open the LongWave Audio flasher: **PLACE HOLDER**
2. Connect the XIAO ESP32-S3 Plus via USB-C.
3. Put the board into download mode: hold `BOOT`, tap `RESET`, then release `BOOT`.
4. Select your board variant and regulatory region:
   - **Board A — Wio-WM6180** (Seeed FGH100M chipset)
   - **Board B — Custom PCB** (OSHWLab mm8108)
   - Region: `AU2020`, `AU`, or `US`
5. Click **Install Firmware**, select the serial port when prompted, and wait for the flash to complete.
6. Press `RESET` (without holding `BOOT`) to boot the new firmware.

Continue to **[Provisioning](#provisioning-esp-softap)** to bring the endpoint online.

---

## Provisioning (ESP SoftAP)

A freshly flashed endpoint comes up as a 2.4 GHz Wi-Fi access point and waits to be told which Wi-Fi HaLow network to join. Provisioning is done from your phone using Espressif's SoftAP Provisioning app — **not** the BLE/QR code variant.

> **Attach the 2.4 GHz antenna to the XIAO before provisioning.** Without it, the SoftAP signal is very weak and you'll need to be right next to the target 2.4 GHz AP for credentials to take.

### Install the App

| Platform | App | Link |
|---|---|---|
| iOS | ESP SoftAP Provisioning | [App Store](https://apps.apple.com/app/esp-softap-provisioning/id1474664106) |
| Android | ESP SoftAP Provisioning | [Google Play](https://play.google.com/store/apps/details?id=com.espressif.provsoftap) |

### Steps

1. Power the endpoint via USB-C. The status LED will slow-blink purple (waiting for network).
2. On your phone, join the endpoint's 2.4 GHz Wi-Fi network. The SSID begins with `PROV_` followed by a device-specific suffix. There is no password.
3. Open the **ESP SoftAP Provisioning** app and tap **Provision New Device**.
4. When prompted, **do not scan a QR code** — choose the option to provision over the existing Wi-Fi connection (e.g. "I don't have a QR code" / "Connect manually").
5. Enter the proof of possession: `longwave`
6. Select your Wi-Fi HaLow SSID from the list (or enter it manually) and provide the HaLow password.
7. Confirm. The endpoint stores the credentials, drops its SoftAP, and joins the HaLow network.

> The ESP SoftAP Provisioning app may report **"failed to associate"** at the end of the flow. This is expected.

The status LED will progress: slow purple blink (provisioning) → slow blue blink (waiting for HaLow) → solid blue (connecting to Snapserver) → 1s green flash (connected). See [Status LED](#status-led).

### Re-provisioning

Provisioned credentials are stored in NVS and persist across reboots. To clear them and bring the SoftAP back up, hold the `RESET` button for **5 seconds** until the status LED flashes **red**. The endpoint will reboot into SoftAP mode and can be re-provisioned from the app.

---

## Build from Source (Developers)

This path is for firmware development and contributing changes upstream. End users should use the [web flasher](#flash-via-web-preferred).

### Prerequisites

- [ESP-IDF v5.4.2](https://docs.espressif.com/projects/esp-idf/en/v5.4.2/esp32s3/get-started/)
- USB-C cable
- An assembled endpoint

### Set Up ESP-IDF

```bash
git clone -b v5.4.2 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
./install.sh esp32s3
source export.sh
```

### Clone the snapclient HaLow Fork

```bash
git clone --recursive https://github.com/abaumann9/snapclient_halow
cd snapclient_halow
```

### Configure

Sane defaults can be selected via `SDKCONFIG_DEFAULTS`:

```bash
SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32s3;sdkconfig.defaults.xaio-audio-hat;sdkconfig.defaults.<halow-board>"
idf.py set-target esp32s3
```

`<halow-board>`:
- Seeed Studio Wio-WM6180: `seeed_halow_xiao_FGH100M_H`
- OSHWLab mm8108 board: `halow_xiao_mm8108`

Then:

```bash
idf.py menuconfig
```

Set your country code:

```
CONFIG_HALOW_COUNTRY_CODE=?
```

Wi-Fi HaLow SSID/password are no longer compiled in by default — they are set at runtime via [SoftAP provisioning](#provisioning-esp-softap). To hardcode them anyway (e.g. for bench testing), set:

```
CONFIG_HALOW_SSID="<network-ssid>"
CONFIG_HALOW_PASSWORD="<network-password>"
```

If mDNS is not [configured on the snapserver](server-setup.md#3-configure-mdns), disable mDNS and set the snapserver location:

```
CONFIG_SNAPSERVER_HOST="<server-ip>"
CONFIG_SNAPSERVER_PORT=1704
```

### Build and Flash

```bash
idf.py build
idf.py -p /dev/ttyACM0 flash
```

### Monitor

```bash
idf.py -p /dev/ttyACM0 monitor
```

Expected boot sequence:

1. ESP32-S3 boot messages
2. Wi-Fi HaLow module initialization
3. SoftAP advertised (if unprovisioned) or HaLow network connection (if provisioned)
4. Snapserver connection attempt
5. Audio stream synchronization (once music is playing)

---

## Status LED

When using the xaio audio board, the status LED indicates device state:

| Colour | Pattern | Meaning |
|--------|---------|---------|
| Off | — | Idle / no state |
| Purple | Slow blink (300ms on / 700ms off) | SoftAP provisioning mode (waiting for credentials) |
| Blue | Slow blink (300ms on / 700ms off) | Waiting for HaLow network |
| Blue | Solid | Connecting to Snapcast server |
| Green | 1s flash then off | Connected |
| Orange | Fast blink (200ms on / 200ms off) | Resyncing audio |
| Red | Flash | Credentials cleared (5s RESET hold) |
| Red | Solid | Error |

---

## Troubleshooting

### SoftAP Network Doesn't Appear

- Verify the endpoint is in provisioning mode (status LED slow-blinking purple). If it's blue, credentials are already stored — clear them via [Re-provisioning](#re-provisioning).
- The SoftAP is 2.4 GHz only — make sure your phone Wi-Fi is enabled and not locked to 5 GHz.
- If the device was previously provisioned, it will not advertise a SoftAP. Re-provision by erasing stored credentials (see [Re-provisioning](#re-provisioning)).

### Provisioning App Rejects Proof of Possession

- The PoP is exactly `longwave` (lowercase, no quotes).
- If you've changed it in a custom firmware build, use that value instead.

### HaLow Module Not Detected

- Check SPI connections between the ESP32-S3 and HaLow module.
- Verify the module is properly seated (Wio-WM6180) or soldered (custom PCB).
- Check that the correct SPI pins are configured in sdkconfig.
- Try running the porting assistant example app in the HaLow component. You will need to update the SDK appropriately.

### Cannot Connect to Snapserver

- Verify the server IP is correct in the configuration.
- Check that Snapserver is running and listening on port 1704.
- Verify the HaLow network is up and the endpoint has an IP address.
- Check firewall rules on the server.

### No Audio Output

- Verify I2S pin assignments match the DAC hat wiring.
- Check DAC hat connections and power.
- Verify the DAC chip is receiving I2S clock signals.
- Test with a known-good audio source.
