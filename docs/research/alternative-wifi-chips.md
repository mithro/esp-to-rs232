# Non-Espressif low-power Wi-Fi chips

Research date: 2026-11-23. This is a desk survey. Only three sources were
re-read directly (nRF7002 electrical spec, SiWG917 current app note,
CC3351MOD DTIM table); the rest were gathered from datasheets and vendor
documents in one pass and have not been double-checked. Nothing was measured.
None of the figures are guaranteed limits: they are datasheet typicals,
vendor eval-kit measurements, marketing figures or estimates, as noted.

The budget is 30 mW average, about 7 mA at 3.3 V. For the Espressif options
see [espressif-sleep-and-tasmota.md](espressif-sleep-and-tasmota.md).

## Summary

* **One part is a step change: the Qualcomm QCC730 / QCC730M.** Its transmit
  burst is about 142 mW (about 43 mA at 3.3 V) and receive about 10 mW. Every
  other part transmits at 180–320 mA and receives at 20–60 mA.
* **Every other "ultra-low-power" part is in the same class as an Espressif
  chip in auto light sleep:** 0.4–1.2 mA idle at DTIM1, with large transmit
  peaks.
* **Almost none can keep their own UART receiving while asleep.** The usual
  answer is a Wi-Fi network processor plus a tiny always-on microcontroller
  that buffers the serial bytes. The one documented single-chip exception is
  the Realtek RTL8721Dx.
* **The Tasmota-like firmware route (OpenBeken on Beken chips) does not meet
  the budget:** roughly 10–25 mA connected.
* **Under traffic, the time the radio stays awake after each transmission
  costs more than the transmission itself.** Batching rate and that linger
  time matter more than chip choice, QCC730 excepted.

## Workload model

All "estimate" figures assume 5 packets a second of about 200 bytes, each
costing its airtime (about 2.3 ms at 1 Mbps, about 0.4 ms at MCS0) plus
10 ms of receive time waiting for the TCP acknowledgement. **The 10 ms is an
assumption** and the largest uncertainty; real stacks often linger 20–200 ms.

## Comparison

Idle is the connected average at DTIM1 / DTIM3 / DTIM10.

| Part | Idle | UART alive while asleep? | TX peak / RX | Estimate, 5 packets/s | Package | Firmware |
|---|---|---|---|---|---|---|
| Qualcomm QCC730M | 277 / 140 / 91 µW (≈ 84 / 42 / 28 µA at 3.3 V), eval kit | Not verified. Continuous listen is only 8.93 mW | ≈ 142 mW at 9.2 dBm / 10.2 mW | ≈ 2.4 mW, excluding CPU-awake cost | Module 12.28 × 14.8 mm or 12.28 × 19.8 mm | CodeLinaro SDK, FreeRTOS, binary Wi-Fi libraries |
| Silicon Labs SiWN917 + helper MCU | ≈ 0.41 / 0.15 / 0.06 mA (computed) | Helper MCU does it | 260 mA (1 Mbps), 230 mA (MCS0) / 13–18 mA low-power chain | ≈ 2–6.5 mA | QFN84 7 × 7 mm, dual row | Closed network-processor firmware |
| Silicon Labs SiWG917 alone | same radio | No: its MCU must stay at 7.6 mA or more | same | ≈ 8 mA idle already | same | same |
| InnoPhase Talaria TWO | 414 / 151 / 57 µA | UART wake "~550us": first character likely lost | 178 mA (1 Mbps), 108 mA (MCS0) / 31–37 mA | ≈ 2.3–4.0 mA | Modules ≥ 12.8 mm wide | SDK under NDA |
| TI CC3135 / CC3120 + helper MCU | 690 µA at DTIM1 | Helper MCU does it | 272 mA; calibration peak 450 mA / 53 mA | ≈ 3.5–6.4 mA | VQFN64 9 × 9 mm | SimpleLink SDK, binary network processor |
| TI CC3235S/SF | 710 µA | No: wake "less than 3ms" | 266 mA / 53 mA | as CC3135, plus helper | VQFN64 9 × 9 mm | as above |
| Nordic nRF7002 + nRF54L15 | 3.47 / 1.12 / 0.34 mA | Host UART ≈ +0.13 mA (forum) | 252 mA (1 Mbps) / 49.6 mA | ≈ 8.0 mA at DTIM1 | two QFNs | nRF Connect SDK; this host is "experimental" |
| Infineon CYW43439 + host | 1.19 / 0.48 mA ("design goals") | Host does it | 320 mA (1 Mbps) / 37–43 mA | ≈ 3–6.4 mA plus host | Murata 1YN 6.95 × 5.15 mm | open driver, firmware blob |
| NXP RW612 | 0.90 / 0.44 / 0.33 mA | MCU with clocks is 4.22 mA | 261 mA; calibration peak 576 mA / 45–63 mA | ≈ 10.6 mA | WLCSP / BGA / QFN | MCUXpresso |
| Realtek RTL8721Dx | DTIM10 "84μA" (marketing) | **Yes, documented:** 64-byte FIFO, to 115200 | not found | cannot estimate | QFN48 6 × 6 mm | OpenBeken, with a UART-TCP driver |
| XRadio XR806 | 580 / 230 / 90 µA (preliminary) | unknown | 180 mA at 16 dBm, 85 mA at 10 dBm / 20 mA | ≈ 3 mA if the MCU can sleep | QFN32 4 × 4 mm | OpenBeken (power save not implemented) |
| Beken BK7238 / BK7231N | ≈ 10–25 mA connected | unknown | 270–304 mA; start-up 500 mA / 73–75 mA | ≈ 18–22 mA | modules | OpenBeken, best supported |
| Renesas DA16200 | sleep floor 3.5 µA | No: its MCU is off, waking is a reboot | 280 mA / 38 mA | not viable for continuous traffic | QFN48 6 × 6 mm | FreeRTOS SDK |

Not suitable or no evidence: Morse Micro (HaLow only), Atmosic (no Wi-Fi
part), Telink TLSR9118 (no current figures), Bouffalo BL602/BL616,
WinnerMicro W600/W800, Microchip WINC1500/WFI32 (too large, too hungry).

## Key sources

* nRF7002: "Average current consumption (2.4GHz,DTIM=1, beacon duration
  3.8 ms) | 3.47 mA". <https://docs.nordicsemi.com/r/bundle/ps_nrf7002/page/chapters/elspec/doc/electrical_specification.html>
* QCC730 release note: "WLAN BMPS DTIM1 … 277.26 | DTIM3 … 139.77 | DTIM5 …
  111.84 | DTIM10 … 91.20" µW; "Listen Current @ 2.4G 8.93" mW; "enabling or
  disabling DTIM power save may occasionally lead to system hangs".
  <https://docs.qualcomm.com/doc/RNO-241114234619/topic/ReleaseNote.html>
* QCC730 transmit: CCK 1M at 9.2 dBm, "142.488" mW.
  <https://docs.qualcomm.com/doc/80-WL730-73/80-WL730-73_REV_AC_QCC730_FM01_3_WLAN_iPA_DVT_Test_Report.pdf>
* SiWx917: "M4 in PS2 Active/Standby/Sleep and NWP in Active/Connected Sleep
  is not supported."
  <https://docs.silabs.com/wiseconnect/latest/wifi-siwx917-ble-low-power-application-note/system-overview>
* Talaria TWO: "DTIM = 1 414 / DTIM = 3 151 / DTIM = 5 97 / DTIM = 10 57 µA".
  <https://web.archive.org/web/20240520160345/https://innophaseiot.com/wp-content/uploads/modules/Talaria-TWO-Module+SoC-Datasheet.pdf>
* CC3235: "(The wake-up time is less than 3ms.)"; "Peak calibration current …
  450 mA". <https://www.ti.com/lit/ds/symlink/cc3235sf.pdf>
* CYW43439: "IEEE Power Save PM1 DTIM1 (Avg.) … 1.19".
  <https://www.farnell.com/datasheets/3758304.pdf>
* RW612: "standby mode Full clock gating 4.22"; "Peak digital pre-distortion
  (DPD) current 576 mA". <https://www.farnell.com/datasheets/4801023.pdf>
* RTL8721Dx: "The portion of the command used to wake up that exceeds the
  FIFO depth (64B) will be lost."
  <https://ameba-aiot.github.io/ameba-iot-docs/RTL8721Dx/en/v1.0/application_note/power_save/src/power_save.html>
* XR806: "WLAN DTIM interval with 32.768kHz XTAL: 580uA@DTIM1, 230uA@DTIM3,
  90uA@DTIM10". <https://file.elecfans.com/web2/M00/35/D6/pYYBAGIu5eyAPurAAAajA3UDBH4793.pdf>
* BK7238 under OpenBeken: "current at 12V goes from ~16mA to ~4mA".
  <https://github.com/openshwprojects/OpenBK7231T_App/issues/2029>
* DA16200: communication with peripherals "only in DPM FFM".
  <https://www.renesas.com/en/document/mas/um-wi-030-da16200-da16600-dpm-user-manual>

## Shortlist

1. **Qualcomm QCC730M.** The only part where both the average and the
   transmit burst are small; the storage capacitor shrinks about six-fold.
   Risks: CPU-awake current and UART behaviour in sleep are unverified; the
   module is 12.28 mm wide; output is about 9.5 dBm; Wi-Fi libraries are
   binary; a power-save hang is listed in the release note; no Tasmota-class
   firmware. Seen at DigiKey for about US$8.25 (search snippet).
2. **Realtek RTL8721Dx.** The only single chip with documented UART wake
   with a FIFO, and it has open firmware (OpenBeken). Risks: no DTIM1/DTIM3
   or transmit current found, so it needs measuring; bare QFN48, no small
   module found.
3. **Network processor plus helper microcontroller.** SiWN917 is best on
   power but was out of stock with a 20-week lead time and is an 84-pin
   dual-row QFN. CC3135 is mature and available but marginal (3.5–6.4 mA,
   450 mA calibration peak).

## Storage capacitor, for scale

A 1 Mbps frame at 260 mA for 2.5 ms is about 0.65 mC, which needs about
2.2 mF for a 0.3 V droop. The QCC730 needs roughly a sixth of that.

## Not verified

* JLCPCB availability of any part; several prices are search snippets.
* QCC730: CPU-active current, UART receive during sleep, and the supply
  conditions behind the µW figures (measured with a 1.8 V I/O rail).
* RTL8721Dx and XR806: real DTIM1 current with the UART listening.
* The 10 ms acknowledgement wait behind every traffic estimate.
* Vendors assume different beacon lengths (1 ms to 3.8 ms), so idle figures
  are not directly comparable; the 1 ms ones are optimistic.
