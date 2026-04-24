# Hardware Guide

This guide covers the hardware components, bill of materials, and assembly instructions for building a LongWave Audio endpoint.

## Bill of Materials (per endpoint)

| Component | Description | Link |
|---|---|---|
| XIAO ESP32-S3 Plus | Microcontroller | [Seeed Studio](https://www.seeedstudio.com/Seeed-Studio-XIAO-ESP32S3-Plus-p-6361.html) |
| Wi-Fi HaLow Module | Wio-WM6180 or custom PCB | See below |
| Audio DAC Hat | Custom PCB (I2S) | See below |
| Enclosure | 3D printed or off-the-shelf | TBD |
| Speaker(s) | Passive or active | User choice |
| Power supply | USB-C or battery | User choice |

**Server-side hardware**: Any Linux-capable device (Raspberry Pi, old laptop, NAS, etc.) with a Wi-Fi HaLow AP or bridge to the HaLow network.

## Microcontroller: Seeed Studio XIAO ESP32-S3 Plus

The [XIAO ESP32-S3 Plus](https://www.seeedstudio.com/Seeed-Studio-XIAO-ESP32S3-Plus-p-6361.html) is the brain of each endpoint.

> The Plus variant is required has it has a pre soldered 40 pin connector that is used for the Audio DAC hat.

## Wi-Fi HaLow Module Options

### Option A: Seeed Studio Wio-WM6180

- Off the shelf component
- [Purchase link](https://www.seeedstudio.com/Wio-WM6180-Wi-Fi-HaLow-Module-for-XIAO-p-6395.html)

### Option B: Custom PCB (OSHWLab design)

- Open hardware design on OSHWLab/EasyEDA
- [Project link](https://oshwlab.com/robcarey/3-0067)

## Audio DAC Hat

The DAC hat is a custom open-hardware PCB that connects to the ESP32-S3 via I2S and provides audio output.

- **Interface**: I2S from ESP32-S3 Plus
- **Open hardware files**:
- **Output type**: 2.5mm headphone jack, stereo, un-amplified

## Assembly

### Endpoint Stack

Each endpoint consists of three boards stacked together:

1. **Bottom**: XIAO ESP32-S3 Plus (microcontroller)
2. **Middle**: Wi-Fi HaLow module (Wio-WM6180 or custom)
3. **Top**: Audio DAC hat

### Assembly Steps

1. Attach the Wi-Fi HaLow module to the XIAO ESP32-S3 Plus
   - For Wio-WM6180: use the snap-on socket connection
   - For custom PCB: solder with standard 2.54mm pitch connectors and trim flush
2. Connect the DAC hat via the header
3. Power via USB-C on the XIAO board
4. Connect speaker(s) to the DAC hat output

### Photos

<!-- TODO: Add photos of completed build to images/hardware-photos/ -->

## Enclosure

<!-- TODO: Add 3D printable enclosure files or recommended off-the-shelf options -->


