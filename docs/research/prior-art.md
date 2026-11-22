# Prior art: wireless serial adapters

Research date: 2026-11-22. Prices are from marketplace listings found by web
search on that date and will vary. Nothing here was bought or tested.

No product or project was found that is port-powered, sits inside the
footprint of a DE-9 connector and runs open firmware.

## ESP-based adapters built around a DE-9

| Design | Chip | Notes | Source |
|---|---|---|---|
| Hackaday DIY adapter (2021) | ESP32 | Attached to a DB-9 plug, true RS-232 levels, Telnet on port 23 | <https://hackaday.com/2021/04/18/diy-wireless-serial-adapter-speaks-true-rs-232> |
| Elektor ESP32-RS-232 adapter | ESP32 | Published design for classic test equipment | <https://www.elektormagazine.com/articles/esp32rs232-adapter-wireless-link> |
| GhostBridge GB-01.1 | ESP32 | MAX3232, male DB9, USB-C power, 3D-printed case | <https://lectronz.com/products/ghostbridge-gb-01-1-wi-fi-to-rs-232-bridge> |
| RetroDisks WiRSa v3 | ESP | Female DB9, SD card, OLED, for retro computers | <https://www.tindie.com/products/retrodisks/wirsa-v3-wifi-rs232-serial-modem-adapter-with-sd> |
| AIRcable SerialWiFi | ESP8266 | Commercial housing; can take power from DB9 pin 9 | <https://aircable.co/shop/product/acc2315-aircable-serialwifi-14> |

## Ready-made Wi-Fi serial servers

| Product | Approx. price | Notes | Source |
|---|---|---|---|
| Elfin-EW10 / EW10A (Hi-Flying) | £15, US$20–34 | TCP/UDP, Modbus; DB9 not confirmed by the listings | <https://www.ebay.de/itm/124185428378>, <https://www.onbuy.com/gb/p/elfin-ew10a-serial-server-supports-multiple-network-protocols-and-remote-management-rs232-to-wifi~p240487572/> |
| USR-W610 (USR IoT) | US$49–107 | Wi-Fi and Ethernet, RS-232 and RS-485 | <https://ebay.com/itm/255795508002> |
| usconverters Serial WiFi Adapter | about US$80 | Male DB9; power from pin 9 or 5–40 V | <https://www.usconverters.com/serial-rs232-wifi-adapter> |

## Bare modules

ESP8266/ESP8285 "serial to Wi-Fi" modules sell for about £2–12
(<https://www.ebay.de/itm/192288955978>). They are 3.3 V logic only, with no
level shifter and no connector.

## Firmware

* esp-link (ESP8266): <https://hackaday.com/2015/09/18/transparent-esp8266-wifi-to-serial-bridge/>
* ESP-Serial-Bridge (ESP32, three UARTs over TCP): <https://github.com/yuri-rage/ESP-Serial-Bridge>

## Why none fit

* All need an external supply (USB, barrel jack, or a non-standard pin 9).
* The ready-made boxes run closed vendor firmware.
* The ESP-based ones use a charge-pump transceiver (MAX3232 or similar), which
  spends power generating voltages the PC's port already provides.
