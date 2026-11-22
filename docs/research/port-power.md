# How much power a COM port can supply

Research date: 2026-11-23. Datasheet figures were read from the documents
linked. Values marked "≈" were read off plotted curves and are approximate.
Nothing here has been measured on a real port yet.

## Summary

| Port type | Raw power from TXD + DTR + RTS | After rectifying and regulating to 3.3 V |
|---|---|---|
| Legacy, 1488-class drivers on ±12 V | about 100–150 mW | plan on 60–100 mW |
| Modern, 3.3 V charge-pump transceiver (MAX3243 class) | about 33 mW guaranteed, 50–75 mW typical | plan on 20–50 mW |

The original estimate of 120 mW holds only for legacy ports. A modern port
gives roughly 6–15 mA at 3.3 V, not 40 mA.

## Driver output under load

| Driver | Output at 3 kΩ | Under heavier load | Short-circuit | Source |
|---|---|---|---|---|
| MC1488 / SN75188 (±9 V rails) | VOH "min 6, typ 7" (at ±13.2 V: "min 9, typ 10.5") | ≈ 8 mA at +4 V, ≈ 10 mA at 0 V; flattens into a current limit | "6 / 9 / 12 mA"; "Current-Limited Output: 10 mA Typical" | <https://www.ti.com/lit/ds/symlink/mc1488.pdf> |
| SN75185 / GD75232 (±9 V) | VOH "6 min, 7.5 typ" | not extracted | "−4.5 / −12 / −19.5 mA" | <https://www.ti.com/lit/ds/symlink/sn75185.pdf> |
| MAX232 (5 V charge pump) | "5 min, 7 typ" | at 1 kΩ: ≈ +5.5 V, ≈ −4 V | "±10 mA" | <https://www.ti.com/lit/ds/symlink/max232.pdf> |
| MAX3243 (3.3 V) | "5 min, 5.4 typ" | VOH ≈ 5.3 V at 5 mA, ≈ 4.95 V at 10 mA, ≈ 4.2 V at 20 mA. VOL ≈ −5.2 V at 5 mA, ≈ −4.75 V at 10 mA, ≈ −3.0 V at 20 mA | "±35 typ, ±60 max mA" | <https://www.ti.com/lit/ds/symlink/max3243.pdf> |
| MAX3232 (3.3 V), both drivers loaded | 5 / 5.4 V | VOH ≈ 5.1 V at 5 mA, ≈ 4.8 V at 9 mA, ≈ 3.6 V at 15 mA. VOL ≈ −5.0 V at 6 mA, ≈ −4.1 V at 10 mA, collapses near 13 mA | "±35 typ / ±60 max mA" | <https://www.ti.com/lit/ds/symlink/max3232.pdf> |
| SP3243E (3.3 V) | "±5.0 min ±5.4 typ" | ≈ +4.7 / −4.0 V at 8.6 mA | "±35 typ ±60 max mA" | <https://maxlinear.com/document?id=23215> |

Points to note:

* The only **guaranteed** multi-milliamp figure for the 3.3 V parts is the
  "mouse driveability" spec. MAX3243: "DIN1 = DIN2 = GND, DIN3 = VCC, 3-kΩ to
  GND at DOUT3, DOUT1 = DOUT2 = 2.5 mA … ±5 V min". That is
  2 × 5 V × 2.5 mA + 5 V × 1.67 mA ≈ 33 mW.
* On charge-pump parts all outputs share one pump, and the **negative rail
  sags first**.
* It is not stated whether the MAX3243 curves load one output or all three.
* Short-circuit figures are not continuous ratings.

## Prior art and field numbers

* Linux `mouse(4)`: "3 TX -12 V, Imax = 10 mA", "4 DTR +12 V, Imax = 10 mA",
  "7 RTS +12 V, Imax = 10 mA". <https://man7.org/linux/man-pages/man4/mouse.4.html>
* B&B Electronics table, "Power available from 3 driver outputs", as compiled
  at <https://www.oocities.org/thaimcu/tectic/rs232power.htm> (second-hand):
  1488 "+5V @ 6.5 mA, 98 mW" and "−5V @ 6.5 mA, 78 mW"; MAX232 "+5V @ 5 mA,
  75 mW" and "−5V @ 3.5 mA, 42 mW"; EIA-562 "+4.5V @ 1.5 mA".
* Same page, RTS unloaded / at 8 mA (unverified by its author): "Pentium 133
  11.5V 7.0V", "286 11.5V 3.6V", "Toshiba Laptop 8.4V 4.0V".
* B&B via eeNews: a typical port-powered converter consumes "50 mW".
  <https://www.eenewseurope.com/en/tips-and-tricks-using-port-powered-converters/>

## Line states at power-on

* **After UART reset all three outputs are negative.** TL16C550D: "Modem
  control register — Master reset — All bits cleared", and SOUT "is set to the
  marking (high) level as a result of master reset". The line drivers invert.
  <https://www.ti.com/lit/ds/symlink/tl16c550d.pdf>
* **Firmware varies.** coreboot asserts both ("assert DTR and RTS so the other
  end is happy", `uart8250io.c`). EDK2's `BaseSerialPortLib16550` puts the
  modem control register "into its reset state of 0x00", leaving them
  negative. Vendor BIOS behaviour is closed source and must be measured.
* **Linux** raises DTR and RTS when the port is opened (`tty_port.c`), and
  sets DTR when a serial console is registered (`serial_core.c`).

So at the moment the BIOS starts printing, the board may have only negative
voltage on all three lines. The harvester must work from that.

## Does 0 V read as idle?

Yes for every receiver checked: a low or open input reads as idle (mark).

| Receiver | Low threshold | High threshold |
|---|---|---|
| MAX3243 (3.3 V) | "0.6 min, 1.1 typ" | "1.6 typ, 2.4 max" |
| SP3243E (3.3 V) | "0.6 min 1.2 typ" | "1.5 typ 2.4 max" |
| MAX232 | "0.8 min 1.2 typ" | "1.7 typ 2.4 max" |
| MC1489 | "0.75 / 1.0 / 1.25" | "1 / 1.3 / 1.5" |

MAX3243: "An open input will result in a high output on ROUT." Input
resistance is "3 / 5 / 7 kΩ".

Driving 0 V for idle and at least +3 V for the other bit value therefore
works, and needs no negative rail on our side.

## Auto-powerdown risk

MAX3243-class parts can switch their drivers off when they see no valid
RS-232 level on any receiver input: "all receiver input voltages are between
–0.3V and 0.3V for more than 30µs". If a motherboard enables this, a board
that only ever drives 0 V turns off its own power source. The cure is to hold
one PC input (for example DSR) beyond ±2.7 V, at a cost of about 1 mA.
Whether any given motherboard enables auto-powerdown is not verified.

## Not verified

* Maxim application notes AN827, AN860 and AN3424 (host unreachable).
* The ST3243 datasheet.
* Which transceiver particular Supermicro, Dell or HP boards fit.
* Vendor BIOS handling of DTR and RTS during console redirection.
* All second-hand forum and B&B figures.
