# ESP32-P4 MIPI DSI bring-up notes (public pitfalls)

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). People keep landing on P4 + MIPI DSI and burning a day on the same handful of traps. **None of this was bench-tested on our panels for this commit** — I'm only collecting what public issues, PRs, and product pages already say, plus pointers into the other notes in this repo.

## Don't hard-code `MIPI_DSI_PHY_CLK_SRC_DEFAULT` on P4 v3.x + IDF 6.1

New P4 silicon is showing up as **revision v3.x** (v3.2 in the logs). On ESP-IDF **6.1**, the enum value `MIPI_DSI_PHY_CLK_SRC_DEFAULT` still points at the **legacy** clock (`PLL_F20M`) that older P4 revisions used. On v3.x that source isn't valid in the clock switch table, so `esp_lcd_new_dsi_bus` just **aborts**. Leave `phy_clk_src` at 0 / unset and let IDF pick for the chip you actually have.

Public trail: [iclforge#1604](https://github.com/iainchesworthlabs/iclforge/pull/1604) (merged fix that stopped writing the DEFAULT enum), and [Waveshare ESP32-display-support#43](https://github.com/waveshareteam/ESP32-display-support/issues/43) — ESP32-P4-Module-DEV-KIT, chip **v3.2**, IDF **6.1.0**, 7-DSI-TOUCH-C / `esp_lcd_ek79007`, backlight over I²C works, **no picture**, even with the esp-bsp display example and the right screen selected in menuconfig. If your log says abort right when the DSI bus is created, check the chip revision line first before you start rewriting lane rates.

## Panel letter is part of the SKU

Waveshare-style names like `7-DSI-TOUCH-C` vs `7-DSI-TOUCH-A` are not cosmetic. Different letters often mean different controller components (`esp_lcd_ek79007` vs `esp_lcd_ili9881c`, etc.), different Pi `dtoverlay=…,4_0_inch_c` strings, and different registry package names. Copying a same-size wiki page with the wrong letter is a classic backlight-on / image-off. Longer write-up: [`PANEL-LETTER-SKU-TRAP.md`](PANEL-LETTER-SKU-TRAP.md).

## JD9365 “panel enable” may be an I²C write, not a GPIO

On Waveshare 10.1-DSI-TOUCH-A style boards the panel enable path people miss is I²C device **0x45**, register **0x95** — without it ESPHome `mipi_dsi` can wedge after ~19 DCS commands into watchdog / safe mode. Details and the public issue: [`JD9365-PANEL-ENABLE-I2C.md`](JD9365-PANEL-ENABLE-I2C.md).

## Long CUSTOM init dumps (>127 data bytes)

ESPHome's mipi_* path historically masked command length with `x & 0x7F`. A CUSTOM init with more than 127 data bytes gets misread or truncated (Malformed / hang). PRs [#21440](https://github.com/esphome/esphome/pull/21440) / [#21441](https://github.com/esphome/esphome/pull/21441) proposed a fix but were closed unmerged last I checked — if your dump is huge, split it the way the datasheet allows. Notes: [`LONG-INIT-127.md`](LONG-INIT-127.md).

## Host-only boards: 2-lane DSI FPC, no panel on the PCB

Not every P4 board ships a glued-on TFT. CNX's write-up of the [LILYGO T-Halow P4](https://www.cnx-software.com/2026/10/23/lilygo-t-halow-p4-a-m-2-2280-board-with-esp32-p4-wifi-halow-and-esp32-c6/) (2026-10-23) lists a **2-lane MIPI DSI connector for external displays, with touch support** and no onboard screen. That path is “bring your own panel”: match lane count, FPC pitch, and touch bus before you paste an all-in-one BSP example that assumes a specific glued SKU. Same class of matrix thinking as [`HOST-COMPAT-MATRIX.md`](HOST-COMPAT-MATRIX.md).

## What I check, in order

Boot log for **chip revision**. If you're on IDF 6.1 and v3.x, don't force `MIPI_DSI_PHY_CLK_SRC_DEFAULT`. Confirm the wiki / component name matches the **exact** panel letter on the silkscreen. If the board docs mention I²C `0x45` for backlight or enable, do that before you chase DSI timing. Only then worry about lane count vs a host-only 2-lane FPC.

## Disclosure

I work for Display Engineer. One link above points at our site; the Waveshare / ESPHome / CNX / iclforge URLs are the primary sources. Nothing in this file was measured on our bench for this commit.
