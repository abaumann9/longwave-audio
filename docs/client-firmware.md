# Client Firmware

This guide covers building and flashing the snapclient firmware to the XIAO ESP32-S3 Plus endpoints.

TODO: Once DPP PB is implemented upload prebuilt binaries so building from source is not required.

## Prerequisites

- [ESP-IDF v5.4.2](https://docs.espressif.com/projects/esp-idf/en/v5.1.1/esp32s3/get-started/) installed and configured
- USB-C cable for flashing
- A built endpoint (see [Hardware Guide](hardware-guide.md))

## Set Up ESP-IDF

```bash
# Clone ESP-IDF v5.4.2
git clone -b v5.4.2 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
./install.sh esp32s3
source export.sh
```

## Clone the snapclient HaLow Fork

```bash
git clone --recursive https://github.com/abaumann9/snapclient_halow
cd snapclient
```

## Configuration

### sdkconfig

Sane defaults are can be configured via `SDKCONFIG_DEFAULTS`

```bash
SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32s3;sdkconfig.defaults.xaio-audio-hat;sdkconfig.defaults.<halow-board>"
idf.py set-target esp32s3
```
halow-board:
 - Seeed Studio Wio-WM6180: seeed_halow_xiao_FGH100M_H
 - OSHWLab mm8108 board:halow_xiao_mm8108

### SDK config
using
```bash
idf.py menuconfig
```

Set your country code:
```
CONFIG_HALOW_COUNTRY_CODE=?
```

Set you wifi SSID and password:
```
CONFIG_HALOW_SSID="<network-ssid>"
CONFIG_HALOW_PASSWORD="<network-password>"
```

If mDNS was not [configured on the snapserver](server-setup.md#3-configure-mdns) disable mDNS and set the snapserver location.
```
CONFIG_SNAPSERVER_HOST="<server-ip>"
CONFIG_SNAPSERVER_PORT=1704
```

If performance is sub-optimal you can disable power management and the tickless
kernel.
```
CONFIG_PM_ENABLE=n
CONFIG_FREERTOS_USE_TICKLESS_IDLE=n
```

Once configured
```bash
idf.py build
```


## Flashing

1. Connect the XIAO ESP32-S3 Plus via USB-C
2. Flash:

```bash
idf.py -p /dev/ttyACM0 flash
```

## First Boot and Verification

After flashing, monitor the serial output:

```bash
idf.py -p /dev/ttyACM0 monitor
```

You should see:

1. ESP32-S3 boot messages
2. Wi-Fi HaLow module initialization
3. HaLow network connection
4. Snapserver connection attempt
5. Audio stream synchronization (once music is playing)

## Status LED

When using the xaio audio board the status LED will tell you the state of the device

| Colour | Pattern | Meaning |
|--------|---------|---------|
| Off | — | Idle / no state |
| Blue | Slow blink (300ms on / 700ms off) | Waiting for network |
| Blue | Solid | Connecting to Snapcast server |
| Green | 1s flash then off | Connected |
| Orange | Fast blink (200ms on / 200ms off) | Resyncing audio |
| Red | Solid | Error |

## Troubleshooting

### HaLow Module Not Detected

- Check SPI connections between the ESP32-S3 and HaLow module
- Verify the module is properly seated (Wio-WM6180) or soldered (custom PCB)
- Check that the correct SPI pins are configured in sdkconfig
- Try running the porting assistant example app in the HaLow component. You will need to update the SDK appropriately

### Cannot Connect to Snapserver

- Verify the server IP is correct in the configuration
- Check that Snapserver is running and listening on port 1704
- Verify the HaLow network is up and the endpoint has an IP address
- Check firewall rules on the server

### No Audio Output

- Verify I2S pin assignments match the DAC hat wiring
- Check DAC hat connections and power
- Verify the DAC chip is receiving I2S clock signals
- Test with a known-good audio source

