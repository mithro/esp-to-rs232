# esp-to-rs232

> [!CAUTION]
> **AI in use, use at own risk.**

A tiny board that plugs straight onto the DE-9 COM port on the back of a PC
and bridges it to Wi-Fi, so the machine's serial console (BIOS, bootloader,
kernel, login) can be read and typed on over the network.

It has no power connector. It runs on what it can take from the port's own
output pins (TXD, DTR, RTS).

This is a sibling of [esp32-to-433mhz](https://github.com/mithro/esp32-to-433mhz)
and [esp32-to-gps](https://github.com/mithro/esp32-to-gps) and follows their
layout.

## Status

| Step | State |
|---|---|
| 1. Research: port power, Wi-Fi chip current, prior art | in progress |
| 2. Design document | not started |
| 3. Schematic and PCB (KiCad, script-generated) | not started |
| 4. Tasmota-based firmware | not started |
| 5. Confirmed on hardware | not started |

## Targets

* **Baud rate:** 9600 is the target the design must meet. 115200 is a
  stretch goal the design should not rule out.
* **Direction:** transmit and receive, with RX and TX swappable.
* **Firmware:** Tasmota preferred, on an ESP32-family or ESP8266 chip. Other
  chips and firmware only if those cannot meet the power budget.
* **Level shifting:** discrete parts, not an off-the-shelf RS-232 transceiver.

## Documents

* [docs/design-notes.md](docs/design-notes.md): requirements and the
  decisions made so far, with open questions.
* [docs/research/port-power.md](docs/research/port-power.md): how much
  power a COM port can supply.
* [docs/research/wifi-mcu-current.md](docs/research/wifi-mcu-current.md):
  what the candidate Wi-Fi chips draw.
* [docs/research/prior-art.md](docs/research/prior-art.md): existing
  wireless serial adapters and why none fit.

## Licence

Apache 2.0. See [LICENSE](LICENSE).
