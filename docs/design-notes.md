# Design notes

Working notes, started 2026-11-22. This records the requirements and the
decisions so far. It is not yet an approved design; the design document comes
after the research is complete.

## What it is for

Reading and typing on a PC's serial console over Wi-Fi: BIOS setup,
bootloader, kernel console, login. A transparent two-way byte bridge with no
protocol of its own.

## Requirements (from the user)

* Plugs directly onto the male DE-9 COM port on the back of a standard PC.
* The PCB has mounting holes for the port's two jack-screws and surface-mount
  sockets that accept the port's pins. It carries no DE-9 connector itself.
* Powered only by rectifying the port's output pins. Estimate: each output
  supplies 5–10 mA at ±12 V, so about 12 V × 10 mA ≈ 120 mW, or roughly
  40 mA at 3 V. **Research says this holds only for legacy ±12 V ports.** A
  modern 3.3 V charge-pump port gives about 20–50 mW after regulation,
  roughly 6–15 mA at 3.3 V. See [research/port-power.md](research/port-power.md).
* **Power target (decided 2026-11-23): 30 mW average draw.** The target
  machines are mostly older servers and x86 motherboards from about
  2015–2020, which use charge-pump transceivers.
* Extra power from a USB port is an absolute last resort. Choosing a
  different Wi-Fi chip and firmware is preferred over that.
* Very small circuit.
* Transmit and receive. 9600 baud is the target; 115200 is nice to have.
* RX and TX directions can be swapped.
* Tasmota preferred, on ESP32 or ESP8266. Other chips or firmware if required.
* No off-the-shelf RS-232 transceiver.
* Serial data will probably need to be batched before sending over Wi-Fi.
* Repository follows esp32-to-433mhz and esp32-to-gps.

## Proposals so far (not yet approved)

### Power

* The port's outputs are TXD (pin 3), DTR (pin 4) and RTS (pin 7).
* TXD idles negative. DTR and RTS are expected to be negative until software
  opens the port, which during POST is never. So the harvester should take
  energy from either polarity, as serial mice did. (The power-on state of DTR
  and RTS is still to be verified.)
* The board should start buffering serial data as soon as it has power,
  because the BIOS prints before Wi-Fi can connect.
* A storage capacitor carries the Wi-Fi transmit bursts; see
  [research/wifi-mcu-current.md](research/wifi-mcu-current.md).

### Level shifting with discrete parts

* **Receive (PC output to chip):** a high-value series resistor into a small
  N-channel MOSFET (BSS138 class, ±20 V gate rating) with a pull-up to 3.3 V.
  Draws microamps. The inversion is undone by the UART's own invert option.
* **Transmit (chip to PC input):** a high-side switch connects the PC's
  receive line to the harvested positive rail for one bit value and lets it
  fall to 0 V for the other, instead of driving it negative. This relies on
  the PC's receiver reading 0 V as idle, which is still to be verified
  against receiver datasheets. The PC's input is a 3–7 kΩ load, so the
  switch draws a milliamp or two only while on.
* **RX/TX swap:** pins 2 and 3 each get both the receive and the transmit
  circuit, and firmware selects which does which.

### Network side

* A raw TCP port using Tasmota's TCP serial bridge, so existing console
  servers (conserver, ser2net) can connect to it.

## Open questions

1. How much power do real ports supply, old (±12 V drivers) and modern
   (3.3 V charge-pump transceivers)? Research in progress.
2. Do DTR and RTS sit negative at power-on, and do BIOSes assert them during
   console redirection?
3. Does 0 V read as idle on the receivers PCs use?
4. Which chip: ESP32-C2, ESP32-C3, ESP32-C6 or ESP8266?
5. Can Tasmota enable the low-power modes the budget needs?
6. Which surface-mount socket accepts DE-9 pins, and what is the exact
   pin and jack-screw geometry?
7. Storage capacitor size and type.
