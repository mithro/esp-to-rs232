# Espressif chips: sleeping with a live UART, and Tasmota

Research date: 2026-11-23. Figures come from the documents linked. Items
marked "inferred" or "estimate" are reasoning from those documents, not
published or measured values.

The question: can an Espressif chip running Tasmota stay connected to Wi-Fi,
keep receiving a 9600 baud UART without losing bytes, and average about 7 mA
at 3.3 V (the 30 mW target after conversion losses)?

## Summary

* **Only the ESP32-C6 can keep its UART receiving through light sleep**, and
  only with ESP-IDF 5.5 or later. The ESP32-C2, ESP32-C3, classic ESP32 and
  ESP8266 lose the first byte or bytes by design.
* **Stock Tasmota cannot reach 7 mA on any of them.** It does not enable
  ESP-IDF power management, and its `Sleep` setting is a polling delay loop.
* **An ESP32-C6 with custom firmware is plausible at about 3.5–4.5 mA idle.**
  That is an estimate assembled from two separate Espressif tables. It has
  not been measured.

| Chip | UART survives light sleep? | Realistic connected average, UART listening | Tasmota |
|---|---|---|---|
| ESP32-C6 | Yes, ESP-IDF ≥ 5.5 | about 3.5–4.5 mA idle (estimate) | `tasmota32c6` exists; no power management |
| ESP32-C2 (ESP8684) | No | 6.6–7.5 mA (must stay in Modem-sleep + DFS) | `tasmota32c2` exists; no power management |
| ESP32-C3 | No | 10–11 mA | yes; no power management |
| Classic ESP32 | No | far over budget | yes; no power management |
| ESP8266 / ESP8285 | No | 15.2–16.2 mA in Modem-sleep | yes |

## UART reception during light sleep

ESP-IDF sleep-modes documentation, "UART Wakeup (Light-sleep Only)":
<https://docs.espressif.com/projects/esp-idf/en/latest/esp32c6/api-reference/system/sleep_modes.html>

* Mode 0: "When all clocks are powered down, the chip can be woken up by
  toggling the RXD pin for a certain number of cycles."
* Mode 1: "Since the UART Core clock remains active, the UART RX can still
  receive data and store it in the RX FIFO. The chip can be woken up from
  Light-sleep when the number of bytes in the RX FIFO exceeds the configured
  threshold."
* Mode 2: "The chip wakes up when the UART RX detects a start bit."

Which chips have which mode (`soc_caps.h` in ESP-IDF master,
<https://github.com/espressif/esp-idf/blob/master/components/soc/esp32c6/include/soc/soc_caps.h>
and its `esp32c2`, `esp32c3`, `esp32` siblings):

* C2, C3 and classic ESP32 have mode 0 only.
* C6 also has the FIFO-threshold, start-bit and character-sequence modes.
  `driver/uart_wakeup.h` is absent at tags v5.3 and v5.4 and present at v5.5.

Mode 0 loses data. ESP-IDF light-sleep example README: "due to limitation of
the HW, the bytes that received during light sleep is only used for waking
up, and it will not be received by UART peripheral or passed to the driver."
<https://github.com/espressif/esp-idf/blob/master/examples/system/light_sleep/README.md>

The other modes need the main crystal running. `uart_wakeup.c` refuses any
other clock source ("failed to setup uart wakeup due to the clock source is
not XTAL!") and keeps the crystal powered in sleep.
<https://github.com/espressif/esp-idf/blob/master/components/esp_driver_uart/src/uart_wakeup.c>

### What keeping the crystal on costs

The only Espressif figures found are from a Bluetooth example:
<https://github.com/espressif/esp-idf/blob/master/examples/bluetooth/nimble/power_save/README.md>

| Chip | Light sleep, main crystal on | Light sleep, 32 kHz crystal |
|---|---|---|
| ESP32-C2 | 2.5 mA | 169 µA |
| ESP32-C3 | 2.3 mA | 140 µA |
| ESP32-C6 | 3.3 mA | 34 µA |

The C6 estimate above is this 3.3 mA plus the 0.17–0.92 mA Wi-Fi figure from
[wifi-mcu-current.md](wifi-mcu-current.md). No source gives a combined
Wi-Fi + crystal-on + UART figure.

### Wake latency

Espressif publishes no "wake to UART ready" time. Default overhead constants
in `components/esp_sleep/sleep_modes.c`: C3 105 + 37 µs, C2 118 + 9 µs,
C6 318 + 56 µs. Powering the flash down adds "approximately 900us"
(`esp_sleep/Kconfig`). One bit at 9600 baud is 104 µs, so waking on the
start bit cannot save the first byte on the C2 or C3 (inferred). On the C6
in mode 1 it does not matter: the 128-byte FIFO takes 133 ms to fill at 9600
baud and 11 ms at 115200.

## ESP32-C6 low-power core and LP UART

* It can receive with the main core in deep sleep: "reading data written to
  a serial console and echoing it back while the main core is in deepsleep."
  <https://github.com/espressif/esp-idf/tree/master/examples/system/ulp/lp_core/lp_uart/lp_uart_echo>
* Pins are fixed: "Only GPIO#4 can be used as the UART Rx pin"; GPIO5 is TX.
* The FIFO is 16 bytes.
* Its default clock is RC_FAST, "17.5 MHz +/- 7%", which is poor for a UART.
* Not verified: receiving on the LP core while the main core is in auto
  light sleep with Wi-Fi connected. An open issue reports problems:
  <https://github.com/espressif/esp-idf/issues/15851>
* The main UART's FIFO-threshold mode is simpler and better documented.

## Tasmota

* **No power management.** The SDK configuration Tasmota builds against has
  no `CONFIG_PM_ENABLE` or `CONFIG_FREERTOS_USE_TICKLESS_IDLE`.
  <https://github.com/Jason2866/esp32-arduino-lib-builder/blob/v55_gcc153_pico/configs/defconfig.common>
  No call to `esp_pm_configure` or `esp_light_sleep_start` was found in
  arendst/Tasmota. Tasmota does allow a per-build `custom_sdkconfig`, so it
  can be compiled in.
* **`Sleep`** on ESP32 is a loop of `delay(1)` that polls every millisecond.
  It is not light sleep.
* **`SetOption60`** on ESP32 only selects the Wi-Fi modem power-save mode.
  Real SDK light sleep is ESP8266-only.
* **`WifiPower`** sets transmit power in dBm (default 17), in steps from
  2 dBm to 19.5 dBm on ESP32.
* **ESP32-C2** is supported: `[env:tasmota32c2]` exists and
  <https://ota.tasmota.com/tasmota32/release/tasmota32c2.bin> is published.
* **TCP bridge** (`xdrv_41_tcp_bridge.ino`): no batching interval. It sends
  up to 255 bytes on every main-loop pass with `setNoDelay(true)`. On ESP32
  it is compiled only into `tasmota32-zbbrdgpro`; other builds need
  `#define USE_TCP_BRIDGE`.
* No measured Tasmota idle current for the C2, C3 or C6 was found.

### What Tasmota on a C6 would need

1. Power management and tickless idle added through `custom_sdkconfig`, and
   a call to `esp_pm_configure()` with light sleep enabled.
2. UART FIFO-threshold wakeup set up on the bridge UART.
3. The 1 ms polling loop replaced by a blocking wait, or auto light sleep
   never triggers (inferred).
4. A batch timer and a larger buffer in the TCP bridge.
5. `CONFIG_ESP_WIFI_SLP_DEFAULT_MIN_ACTIVE_TIME` reduced from its 50 ms
   default (range 8–60 ms). At 78 mA receive current, 50 ms per batch is
   about 3.9 mC; two batches a second would be 7.8 mA on their own. At 8 ms
   it is about 0.6 mC per batch (estimate from the Kconfig text and the
   datasheet receive figure).

With those changes Tasmota contributes little beyond its configuration
interface. A small dedicated ESP-IDF firmware may be less work.

## ESP8266 / ESP8285

"ESP8266 Low Power Solutions" v1.5, table 1-1:
<https://www.espressif.com/sites/default/files/documentation/9b-esp8266-low_power_solutions__en.pdf>

| DTIM | Modem-sleep | Light-sleep |
|---|---|---|
| 1 | 16.2 mA | 1.8 mA |
| 3 | 15.4 mA | 0.9 mA |
| 10 | 15.2 mA | 0.55 mA |

The UART does not survive light sleep: "The hardware wake-up process lasts
for approximately 3 ms", which is about three bytes at 9600 baud.

## Peak currents

| Chip | TX 802.11b 1 Mbps | TX 802.11n HT20 MCS7 | RX HT20 | Source |
|---|---|---|---|---|
| ESP32-C2 | 370 mA @22 dBm | 300 mA @19 dBm | 65 mA | <https://www.espressif.com/sites/default/files/documentation/esp8684_datasheet_en.pdf> |
| ESP32-C3 | 335 mA @21 dBm | 276 mA @18.5 dBm | 84 mA | <https://documentation.espressif.com/esp32-c3_datasheet_en.pdf> |
| ESP32-C6 | 354 mA @21 dBm | 280 mA @18.5 dBm | 78 mA | <https://www.espressif.com/sites/default/files/documentation/esp32-c6_datasheet_en.pdf> |

The datasheets give Wi-Fi transmit current only at maximum power. The only
power-versus-current data is for Bluetooth LE. C6: 315 mA at 20 dBm, 190 mA
at 9 dBm, 130 mA at 0 dBm. C2: 320 mA at 20 dBm, 190 mA at 9 dBm, 150 mA at
0 dBm, 90 mA at −15 dBm.

All three datasheets recommend a supply of "500 mA or more". Espressif gives
no guidance for sizing a reservoir on a weaker supply.

## Wi-Fi 6 target wake time (C6)

No usable measured figure was found. A wake interval of many seconds would
not suit 100–500 ms batching anyway.

## Not verified

* The 3.5–4.5 mA C6 idle estimate. It should be measured on a C6 with
  crystal-on auto light sleep and Wi-Fi connected before the design depends
  on it.
* Wi-Fi transmit current at reduced power.
* Forum reports of extra byte loss on the C3 (the forum blocked fetching).
