# Wi-Fi chip current consumption

Research date: 2026-11-23.

The question: which Wi-Fi chips can stay connected to an access point on the
power a COM port supplies? The working estimate for that supply is about
120 mW (roughly 40 mA at 3 V); see [design-notes.md](../design-notes.md). The
supply side is not yet verified.

Tasmota runs only on Espressif chips, so those are examined first.

## Average current while connected

Source: Espressif ESP-IDF Programming Guide, "Introduction to Low Power Mode
in Wi-Fi Scenarios", `latest` version, fetched 2026-11-23:

* <https://docs.espressif.com/projects/esp-idf/en/latest/esp32c2/api-guides/low-power-mode/low-power-mode-wifi.html>
* <https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-guides/low-power-mode/low-power-mode-wifi.html>
* <https://docs.espressif.com/projects/esp-idf/en/latest/esp32c6/api-guides/low-power-mode/low-power-mode-wifi.html>

Table header on all three pages:
`Item | Modem-sleep | Modem-sleep+DFS | Auto Light-sleep | Deep-sleep`.
In the first three modes the Wi-Fi connection is maintained; in Deep-sleep it
is dropped. DFS is dynamic frequency scaling of the CPU clock.

| Chip | DTIM | Modem-sleep | Modem-sleep + DFS | Auto Light-sleep |
|---|---|---|---|---|
| ESP32-C2 | 1 | 13.6 mA | 7.5 mA | 1.6 mA |
| ESP32-C2 | 3 | 13.0 mA | 6.9 mA | 0.71 mA |
| ESP32-C2 | 10 | 12.9 mA | 6.6 mA | 0.39 mA |
| ESP32-C3 | 1 | 21.47 mA | 11.35 mA | 1.4 mA |
| ESP32-C3 | 3 | 20.82 mA | 10.71 mA | 0.62 mA |
| ESP32-C3 | 10 | 20.67 mA | 10.32 mA | 0.31 mA |
| ESP32-C6 | 1 | 27.48 mA | 14.37 mA | 0.919 mA |
| ESP32-C6 | 3 | 27.08 mA | 13.9 mA | 0.368 mA |
| ESP32-C6 | 10 | 26.85 mA | 13.71 mA | 0.172 mA |

Notes on these figures:

* The pages say: "All currents in the table are average currents, and the
  measured data are obtained by testing in a shielded box."
* The pages do **not** state the supply voltage, the access point, the beacon
  interval, the transmit power or the traffic. Treat them as best-case.
* The ESP32-C2 summary table gives no unit; the detailed table on the same
  page labels the column "Average current (mA)". CPU frequency is 120 MHz for
  the C2 and 160 MHz for the C3 and C6.
* The pages also give a "Max current (mA)" column, not otherwise defined.
  With DFS on it is 65–66 mA (C2), 81–83 mA (C3) and 83–84 mA (C6) in
  Modem-sleep, and 65–68 mA (C2), 86–87 mA (C3) and 102–103 mA (C6) in Auto
  Light-sleep.
* The ESP32-C6 page gave no figure for Wi-Fi 6 target wake time.

## Peak current (ESP32-C3)

Source: ESP32-C3 Series Datasheet v2.5, section 5.6,
<https://documentation.espressif.com/esp32-c3_datasheet_en.pdf>, fetched
2026-11-23. Conditions, quoted: "The current consumption measurements are
taken with a 3.3 V supply at 25 °C of ambient temperature at the RF port. All
transmitters' measurements are based on a 100% duty cycle."

Table 5-7, Wi-Fi current consumption depending on RF modes:

| Mode | Description | Peak |
|---|---|---|
| TX | 802.11b, 1 Mbps, @21 dBm | 335 mA |
| TX | 802.11g, 54 Mbps, @19 dBm | 285 mA |
| TX | 802.11n, HT20, MCS7, @18.5 dBm | 276 mA |
| TX | 802.11n, HT40, MCS7, @18.5 dBm | 278 mA |
| RX | 802.11b/g/n, HT20 | 84 mA |
| RX | 802.11n, HT40 | 87 mA |

Table 5-8, Modem-sleep (typical, all peripheral clocks disabled / enabled):

| CPU frequency | CPU state | Disabled | Enabled |
|---|---|---|---|
| 160 MHz | running | 23 mA | 28 mA |
| 160 MHz | idle | 16 mA | 21 mA |
| 80 MHz | running | 17 mA | 22 mA |
| 80 MHz | idle | 13 mA | 18 mA |

Table 5-9, low-power modes (typical): Light-sleep 130 µA, Deep-sleep 5 µA,
power off 1 µA.

## What this means

* **The average fits.** Modem-sleep with DFS (7–14 mA) is well under a 40 mA
  supply, if the port really delivers that.
* **The peaks do not.** A transmit burst is about eight times the supply. A
  storage capacitor must carry each burst, and transmit power should be
  reduced in firmware.
* **Serial data must be batched.** Each transmission is a burst of a few
  hundred mA, so bytes are collected and sent together. At 9600 baud the port
  produces at most 960 bytes per second. The batching interval trades latency
  against power.
* **The ESP32-C2 has the lowest awake current of the three.**

## Not yet verified

* Whether a UART can receive at 9600 baud without losing the first byte when
  the chip is in Auto Light-sleep. Until checked, the budget uses the
  Modem-sleep + DFS row.
* Whether a Tasmota build can enable DFS or Auto Light-sleep.
* Transmit current at reduced transmit power.
* Peak figures for the ESP32-C2 and ESP32-C6 from their datasheets.
* Non-Espressif chips (only relevant if Tasmota is given up).
