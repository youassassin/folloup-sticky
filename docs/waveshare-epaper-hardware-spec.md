# Waveshare ESP32-S3-ePaper-3.97 Hardware Specification

> This document describes the hardware this firmware runs on: the main
> controller, key ICs, I2C addresses, power control, and ESP32-S3 GPIO
> definitions. Vendor wiki: <https://docs.waveshare.com/ESP32-S3-ePaper-3.97>.
>
> **Sources of truth.** Pin values come from
> [`components/board/include/waveshare_board_config.h`](../components/board/include/waveshare_board_config.h)
> and PMIC setup from
> [`components/board/waveshare_board.cpp`](../components/board/waveshare_board.cpp).
> Where the vendor wiki and the firmware disagree, the firmware is authoritative
> for what the running code actually drives. The board schematic and the ESP32-S3
> Technical Reference Manual are in this folder as PDFs with text extractions
> (`ESP32-S3_e-Paper-3.97-schematic.*`, `esp32-s3_technical_reference_manual_en.*`).
> The schematic text is noisy because net labels overlap in the PDF, so use the PDF
> itself when you need to read a connection.

## 1. System Overview

| Item | Specification |
|---|---|
| Board | Waveshare ESP32-S3-ePaper-3.97 |
| Main controller | ESP32-S3-WROOM-1-N16R8 (`esp32s3` target) |
| Flash / PSRAM | 16 MB flash / 8 MB octal PSRAM |
| Wireless | 2.4 GHz Wi-Fi (802.11 b/g/n) + Bluetooth 5 LE |
| Display | Waveshare 3.97" e-paper panel, `SSD1677` controller, `800x480` |
| Display bus | SPI (`SPI3_HOST`), write-only, dedicated to the panel |
| Audio codec | ES8311 over I2S + shared I2C |
| Audio amplifier | NS4150B (enabled via `GPIO39`) |
| Audio format | `16 kHz`, mono, 16-bit, full duplex (capture and playback share one clock) |
| Storage expansion | MicroSD via `SDMMC` 4-bit mode |
| Power management | AXP2101 PMIC (the wiki labels it `TG28`) |
| RTC | PCF85063 |
| IMU | QMI8658 (6-axis) |
| Temp/humidity | SHTC3 (present on board; not driven by this firmware) |
| USB | Native `USB-OTG` over the board USB-C connector |
| Shared control bus | I2C on `GPIO41` (SDA) / `GPIO42` (SCL) |

## 2. Key IC Part Numbers

| Function | Part Number | I2C Address | Software-Relevant Notes |
|---|---|---|---|
| Main controller | ESP32-S3-WROOM-1-N16R8 | — | ESP-IDF target `esp32s3`; 16 MB flash / 8 MB PSRAM |
| E-paper controller | SSD1677 | — | 800x480 panel; SPI command interface |
| Audio codec | ES8311 | `0x30` | I2S audio + I2C control; PA enable on `GPIO39` |
| Audio amplifier | NS4150B | — | Speaker power amp; enable driven by codec PA pin `GPIO39` |
| Power management IC | AXP2101 | `0x34` | Battery charging + system power rails + power key + VBUS state |
| RTC | PCF85063 | `0x51` | Real-time clock; interrupt line on `GPIO45` (not used by firmware) |
| 6-axis IMU | QMI8658 | `0x6B` | Fixed address in the board config; INT2 on `GPIO40` |
| Temp/humidity sensor | SHTC3 | `0x70` | On shared I2C bus; no driver in this firmware |
| MicroSD | — | — | `SDMMC` 4-bit socket; no card-detect or power-enable line |

## 3. I2C Peripheral Addresses

All I2C peripherals share the same master bus (`GPIO41` SDA / `GPIO42` SCL).

| Peripheral | Part Number | Address | Notes |
|---|---|---|---|
| Audio codec | ES8311 | `0x30` | `ES8311_CODEC_DEFAULT_ADDR` |
| PMIC | AXP2101 | `0x34` | `WAVESHARE_AXP2101_I2C_ADDR` |
| RTC | PCF85063 | `0x51` | `WAVESHARE_PCF85063_I2C_ADDR` |
| 6-axis IMU | QMI8658 | `0x6B` | `WAVESHARE_QMI8658_I2C_ADDR` |
| Temp/humidity sensor | SHTC3 | `0x70` | On the bus per board design; not accessed by firmware |

The bus runs at 400 kHz on `I2C_NUM_1`.

## 4. ESP32-S3 GPIO Definitions

### 4.1 E-paper Display (SPI3)

| Signal | GPIO | Direction (from controller perspective) | Purpose |
|---|---:|---|---|
| `EPD_DC` | GPIO9 | Output | E-paper D/C select |
| `EPD_CS` | GPIO10 | Output | E-paper SPI CS |
| `EPD_SCK` | GPIO11 | Output | E-paper SPI CLK |
| `EPD_MOSI` | GPIO12 | Output | E-paper SPI MOSI (DIN) |
| `EPD_RST` | GPIO46 | Output | E-paper reset |
| `EPD_BUSY` | GPIO3 | Input | E-paper busy status |

There is no MISO line and no GPIO power-enable: the panel is fed by the AXP2101
rails.

### 4.2 Audio (I2S + Codec)

| Signal | GPIO | Direction (from controller perspective) | Purpose |
|---|---:|---|---|
| `I2S_MCLK` | GPIO13 | Output | Codec master clock |
| `I2S_BCLK` | GPIO14 | Output | Codec bit clock |
| `I2S_WS` (LRCK) | GPIO47 | Output | Codec word select / LR clock |
| `I2S_DOUT` | GPIO48 | Output | Codec data out (playback) |
| `I2S_DIN` | GPIO21 | Input | Codec data in (mic capture) |
| `CODEC_PA` | GPIO39 | Output | Codec power-amplifier enable |
| `CODEC_I2C_SDA` | GPIO41 | I/O | Codec control I2C SDA (shared bus) |
| `CODEC_I2C_SCL` | GPIO42 | Output | Codec control I2C SCL (shared bus) |

### 4.3 I2C And Interrupts

| Signal | GPIO | Direction (from controller perspective) | Purpose |
|---|---:|---|---|
| `I2C_SDA` | GPIO41 | I/O/Open-drain | Shared I2C SDA (codec/PMIC/RTC/IMU) |
| `I2C_SCL` | GPIO42 | Output/Open-drain | Shared I2C SCL (codec/PMIC/RTC/IMU) |
| `PMIC_IRQ` | GPIO38 | Input | AXP2101 interrupt (power key; also a light-sleep wake source) |
| `RTC_INT` | GPIO45 | Input | PCF85063 RTC interrupt (not used by firmware) |
| `QMI8658_INT2` | GPIO40 | Input | IMU interrupt 2 (not used; auto-sleep polls the IMU) |

### 4.4 Buttons And Navigation

| Signal | GPIO | Direction (from controller perspective) | Purpose |
|---|---:|---|---|
| `BOOT_BUTTON` | GPIO0 | Input | BOOT/strap pin, also the primary action button |
| `NAV_BUTTON_UP` | GPIO4 | Input | Navigation up |
| `NAV_BUTTON_FUNCTION` | GPIO5 | Input | Navigation function / middle key |
| `NAV_BUTTON_DOWN` | GPIO6 | Input | Navigation down |

All four are active-low to GND. The `PWR` key is not a GPIO; see section 5.3.

### 4.5 MicroSD (SDMMC 4-bit)

| Signal | GPIO | Direction (from controller perspective) | Purpose |
|---|---:|---|---|
| `SD_D0` | GPIO15 | I/O | SD data 0 |
| `SD_D1` | GPIO7 | I/O | SD data 1 |
| `SD_D2` | GPIO8 | I/O | SD data 2 |
| `SD_D3` | GPIO18 | I/O | SD data 3 |
| `SD_CLK` | GPIO16 | Output | SD clock |
| `SD_CMD` | GPIO17 | I/O | SD command |

The schematic labels these nets with their SPI-mode names (`SD_MOSI`, `SD_MISO`,
`SD_CS`, `SD_CLK`) alongside `SD_D1` / `SD_D2`; the firmware drives them as a
4-bit SDMMC bus, not as SPI. The firmware enables the ESP32-S3 internal pull-ups
on these lines.

## 5. Power and Charging (AXP2101)

The AXP2101 PMIC owns system power, battery charging, the power key, and USB VBUS
state. The firmware profile is applied in `components/board/waveshare_board.cpp`
and is the first hardware step in `app_shell::Run()`.

### 5.1 Power Rails

PMIC outputs the firmware enables at bring-up:

| Rail | State | Voltage | Notes |
|---|---|---|---|
| `DC1` | enabled | `3300 mV` | Main system 3.3 V |
| `ALDO1` | enabled | `3300 mV` | 3.3 V peripheral LDO |
| `ALDO2` | enabled | `3300 mV` | 3.3 V peripheral LDO |
| `ALDO3` | enabled | `3300 mV` | 3.3 V peripheral LDO |
| Button/backup battery | charge enabled | — | Coin/backup cell charge (e.g. RTC backup) |
| System power-down | — | `2800 mV` | PMIC cuts system power below this |

> Per-rail **load assignment** (which LDO feeds the display, codec, SD, or RTC) is
> a schematic-level fact and is not declared per rail in firmware: the code only
> sets voltages and enables. Confirm the rail-to-peripheral mapping against the
> board schematic before repurposing any rail.

### 5.2 Charging And VBUS

| Parameter | Value |
|---|---|
| VBUS voltage limit | `4.36 V` |
| VBUS current limit | `900 mA` |
| Charge target voltage | `4.2 V` (single-cell Li-ion/LiPo) |
| Charge constant current | `400 mA` |
| Precharge current | `75 mA` |
| Termination current | `25 mA` |
| System power-down voltage | `2800 mV` |

The firmware does not set the AXP2101's low-battery warning or thermal thresholds,
so those stay at the PMIC defaults.

### 5.3 Power Key And Shutdown Model

The `PWR` key is wired to the AXP2101, not to a GPIO. It reaches the firmware as
PMIC interrupts on `GPIO38`, decoded by `main/power_key_runtime`:

| Press | PMIC setting | Firmware action |
|---|---|---|
| Short press | short-press IRQ | toggle the lock screen |
| Hold, about 1 s | long-press IRQ (`IrqLevelTime` = 1 s) | open the shutdown confirmation |
| Hold, 6 s | power-key press-off time = 6 s | PMIC cuts the rails, no firmware involved |

Only the power-key IRQs are enabled. VBUS insert/remove IRQs are deliberately left
off: nothing consumes them, and the PMIC IRQ line is a light-sleep wake source, so
every cable event would wake the device.

Shutdown model (hybrid):

- power on from the PMIC hardware key path
- firmware-confirmed shutdown before PMIC power-off for the normal in-app flow
  (the confirmation modal, then `power_service::RequestShutdown()`, which clears
  the PCF85063 alarm/timer interrupt sources and calls `Axp2101::PowerOff()`)
- PMIC-enforced `6 s` power-key hold as a forced fallback

While VBUS is present the PMIC keeps the rail fed, so a software power-off can
return with the board still powered; unplugging USB completes it.

## 6. Sleep and Wake-Up

The firmware uses **light sleep** (not deep sleep) for inactivity, driven by
`device_sleep_service`. The product wiring is in `main/device_sleep_runtime.cpp`.

Entry sequence before `esp_light_sleep_start()`:

- suppress display refresh and put the e-paper panel to sleep
- arm GPIO wake sources, then sleep

### 6.1 Wake Sources

| Signal | GPIO | Trigger | Purpose |
|---|---:|---|---|
| `PMIC_IRQ` (power key) | GPIO38 | Low level | Power-key press / PMIC event wake |
| `BOOT_BUTTON` | GPIO0 | Low level | Primary action button wake |

Both are armed with `gpio_wakeup_enable(..., GPIO_INTR_LOW_LEVEL)` plus
`esp_sleep_enable_gpio_wakeup()`. On exit the firmware reads the `BOOT` pin level
to attribute the wake cause, restores the button input before anything slow runs,
and suppresses the press event that caused the wake so it cannot start a
recording. It then disarms the wake sources and recovers the display, the SD card,
and Wi-Fi (see 6.2).

### 6.2 Notes

- This is **light sleep**, so RAM and most peripheral state are kept. There is no
  EXT1 / RTC-domain deep-sleep wake path in this firmware.
- The **SD card does not keep its state** across light sleep. On wake,
  `storage_service::RecoverAfterLightSleep()` unmounts and remounts the card;
  without that the first read times out. The display (`display_service`) and
  Wi-Fi (`wifi_service`) have matching recovery hooks, all called from
  `RestoreAfterLightSleep()`.
- Motion is **not** a wake source. Auto-sleep polls QMI8658 samples to detect
  inactivity; the IMU interrupt line is unused.
- The RTC interrupt (`GPIO45`) is not used as a wake GPIO.

## 7. Strapping Pins

The ESP32-S3 strapping pins carry startup risk; several are reused as functional
signals on this board and must keep a boot-safe default level at reset:

| GPIO | Strap role | Board use | Caution |
|---|---:|---|---|
| `GPIO0` | Boot / download select | Primary action button (also power wake) | Must be high at reset for normal boot |
| `GPIO3` | JTAG source select | `EPD_BUSY` input | Keep default level boot-safe at startup |
| `GPIO45` | VDD_SPI voltage | `RTC_INT` input | Avoid asserting during reset |
| `GPIO46` | Boot / ROM messaging | `EPD_RST` output | Avoid asserting during reset |

## 8. USB / OTG Storage

- OTG mode uses the ESP32-S3 native USB device path (TinyUSB MSC) to expose the SD
  card as USB mass storage.
- The user enters it from the Settings page with the `Enable OTG` button, which is
  a toggle. It is refused with a "No USB connection" modal when no cable is
  attached, with a "Finish the recording first" toast during a recording session,
  and with a no-SD modal when no card is present.
- While active, the SD card is host-owned: the app has no filesystem, and a modal
  stays up with a single `Disable OTG mode` action.
- The only ways out are that modal action or the Settings toggle. Removing the
  cable does not exit on its own, because the VBUS interrupts are not enabled (see
  section 5.3).
- Exiting tears USB down and always remounts the card, even if teardown reports an
  error.

## 9. Notes For Working On The Hardware

| Item | Notes |
|---|---|
| Source of truth | Pin values come from `waveshare_board_config.h`. |
| Shared I2C bus | Codec, PMIC, RTC, IMU, and SHTC3 share one master bus on `GPIO41`/`GPIO42`. |
| IMU address | QMI8658 is used at `0x6B`; there is no alternate-address fallback in the firmware. |
| Strapping pins | `GPIO0/3/45/46` are strapping pins reused as functional signals; see [Strapping Pins](#7-strapping-pins). |
| SD mode | `SDMMC` 4-bit (not SPI); all four data lines connected (`D0`-`D3`). No card-detect, so a missing card shows up as a failed mount. |
| Audio sample rate | The codec runs at `16 kHz` mono/16-bit for both capture and playback, matching the recording and Gemini pipeline, so nothing resamples. Playback volume is set to full scale because the amp and small speaker have no headroom to spare. |
| PMIC naming | The Waveshare wiki lists the PMIC as `TG28`; the firmware drives an AXP2101-compatible PMIC at I2C `0x34`, and that is what works. Treat `0x34` / AXP2101 as authoritative. |
| SHTC3 sensor | Present on the board (shared I2C, `0x70`) but the firmware ships no SHTC3 driver; add one before relying on temp/humidity. |
| Connectors | Battery, speaker, and RTC-backup-battery use MX1.25 headers (per vendor wiki); USB-C is used for flashing/logging and native USB-OTG. |
| Console | The USB-C port carries the USB-Serial-JTAG console (`CONFIG_ESP_CONSOLE_SECONDARY_USB_SERIAL_JTAG`). |

## 10. Build And Flash

The project is an ESP-IDF project (v5.4) targeting `esp32s3`:

```bash
source $IDF_PATH/export.sh
idf.py set-target esp32s3
idf.py build
idf.py flash monitor
```

Project defaults live in `sdkconfig.defaults` (16 MB QIO flash, octal PSRAM, OTA
rollback, TinyUSB MSC); the generated `sdkconfig` is not tracked. The partition
table is `partitions.csv`, a two-slot OTA layout across the 16 MB flash.
