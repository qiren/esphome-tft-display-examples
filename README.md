# ESPHome TFT display examples

ESPHome [`mipi_spi`](https://esphome.io/components/display/mipi_spi/) example configs and bring-up
notes for small TFT LCD modules (ST7789V, GC9A01, ...).

> **Disclosure:** this repository is maintained by [Display Engineer](https://www.displayengineer.com/),
> the manufacturer of the modules listed below (modules built by Shenzhen PengJi Photoelectricity Co., Ltd.).
> The configs are plain ESPHome and should work with any panel that uses the same controller,
> resolution and wiring.

> **Status: not yet verified on hardware.** The configs are written from the ESPHome `mipi_spi`
> documentation and the public product pages. Every value marked `# TESTING-NEEDED` comes from
> docs/datasheet defaults and has **not** been confirmed on a real panel yet. Check them on your
> hardware; this README will be updated once each config has been bench-tested.

## Supported modules

| Module | Size / resolution | Panel | Controller | Interface | Config | Status |
|---|---|---|---|---|---|---|
| [PJ2801H22-23H10P200](https://www.displayengineer.com/products/2-8-inch-240x320-tft-lcd-spi-pj2801h22-23h10p200) | 2.8", 240×320 | TN | ST7789V | SPI | [`esphome/pj2801h22-st7789v-esp32s3.yaml`](esphome/pj2801h22-st7789v-esp32s3.yaml) | TESTING-NEEDED |

### PJ2801H22-23H10P200 specs (from the product page)

2.8", 240×320, TN, SPI, ST7789V driver, pixel pitch 0.180 × 0.180 mm, 200 cd/m² typical,
6 o'clock viewing direction, 65K colours, outline 49.90 × 69.10 × 2.50 mm,
active area 43.20 × 57.60 mm, 4-LED backlight, no touch. Native orientation is portrait (240×320).

FPC pin-out, supply voltage and backlight current are in the datasheet (request it on the
[product page](https://www.displayengineer.com/products/2-8-inch-240x320-tft-lcd-spi-pj2801h22-23h10p200));
they are not repeated here until confirmed.

## Wiring (example, ESP32-S3-DevKitC-1)

The GPIOs are an example; change them in the YAML to match your board. Check the module's FPC
pin-out in the datasheet before connecting anything.

| Panel signal | ESP32-S3 GPIO | Notes |
|---|---|---|
| SCL / SCK | GPIO12 | SPI clock |
| SDA / MOSI | GPIO11 | SPI data |
| CS | GPIO10 | Chip select |
| DC (RS) | GPIO9 | Data/command, required for 4-wire SPI |
| RESET | GPIO8 | |
| Backlight | GPIO14 | Through a transistor/LED driver; LED current per datasheet (TESTING-NEEDED) |
| VCC / GND | per datasheet / GND | Logic supply voltage: see datasheet (TESTING-NEEDED) |

## Quick start

1. Wire the panel as above (or edit the pins in the YAML).
2. Copy [`esphome/pj2801h22-st7789v-esp32s3.yaml`](esphome/pj2801h22-st7789v-esp32s3.yaml) into your ESPHome config folder.
3. Flash it: `esphome run pj2801h22-st7789v-esp32s3.yaml`
4. You should see three horizontal bars: **red** (top), **green**, **blue** (bottom).
   If not, use the table below.
5. Once the bars are correct, raise `data_rate` step by step (e.g. 20 MHz, 40 MHz) and keep the
   highest rate that stays clean on your wiring.

`ST7789V` is a built-in `mipi_spi` model, so no custom `init_sequence` is needed. Keep the native
240×320 portrait dimensions and use `rotation:` for landscape; do not swap width/height by hand
(see [esphome#17074](https://github.com/esphome/esphome/issues/17074)).

## Troubleshooting

| Symptom | Likely cause | What to try |
|---|---|---|
| **White screen**, backlight on | Panel never received a valid init sequence | Check DC, CS and RESET wiring; confirm `model: ST7789V`; lower `data_rate` (e.g. 10 MHz or less); check supply voltage. See the [white screen guide](https://www.displayengineer.com/guides/tft-display-white-screen-causes-fixes). |
| **Wrong colours**: bars look like a negative (cyan / magenta / yellow) | Colour inversion | Flip `invert_colors` |
| **Wrong colours**: top bar blue, bottom bar red | Channel order | Set `color_order: rgb` (default is `bgr`) |
| **Mirrored** or upside-down image | Orientation | Try `rotation: 180` first; if the image is truly mirrored, use `transform:` (`mirror_x` / `mirror_y`, requires `dimensions:`) as described in the [mipi_spi docs](https://esphome.io/components/display/mipi_spi/) |
| **Colour bars** shifted, or a garbage strip on one edge | Offset / window mismatch | Set `dimensions:` explicitly with `offset_width` / `offset_height` (and `pad_*` if needed) |
| Bars appear but flicker or show noise at higher speed | Signal integrity | Shorter wires or a proper FPC adapter; lower `data_rate` |
| Nothing at all, backlight off | Backlight not driven | Check the backlight driver circuit and GPIO polarity |

## Moving off ili9xxx / st7789v / st7735

ESPHome removes those platforms in 2027.5.0 (PRs #20687 to #20690). [`MIGRATING-to-mipi_spi.md`](MIGRATING-to-mipi_spi.md) has before/after YAML for ILI9341, ST7789V, ST7735, GC9A01A and ILI9488, plus what to do with ILI9163 (`CUSTOM`). Like the rest of this repo, it hasn't been tested on hardware yet.

## ESPHome 2026.10.0 display notes

[`ESPHOME-2026.10-display-notes.md`](ESPHOME-2026.10-display-notes.md) covers the 2026.10.0 fixes that matter for TFT/RGB panels: ST7701S black after reboot (50 ms reset delay, #17646) and `rotation: 180` on pure RGB `mipi_rgb` panels (#20757).

## Not sure which controller you have?

[`IDENTIFY-CONTROLLER.md`](IDENTIFY-CONTROLLER.md) shows how to read the ID over SPI before picking a `mipi_spi` model: ST7789V answers `85 85 52` (04h, or DAh/DBh/DCh one byte at a time), some ST7789P3 boards answer `81 81 b3` (same family), ILI9341 answers `00 93 41` on D3h.

[`TOUCH-IC-NOTES.md`](TOUCH-IC-NOTES.md) covers the Waveshare 2.8" CST328→CST3530 silent swap, GT911 I2C `0x5D` vs `0x14`, and ICNT86 default `0x48` on the Waveshare 2.90″ e-paper Touch HAT.

Controller datasheet pages on the site are growing too — [`ILI9881C`](https://www.displayengineer.com/driver-ic/ili9881c) is live if you landed here from an MIPI DSI / ESP32-P4 board note.

[`VERIFY-TOUCH-IC-I2C.md`](VERIFY-TOUCH-IC-I2C.md) is the bring-up checklist: I2C scan first, then match GT911 `0x5D`/`0x14` (and the RST+INT latch) or CST3530 `0x58` before editing YAML. Also covers the "touch reports ~74×70" false-geometry trap (often wrong I²C address / wrong chip, not a calibration problem).

## Round GC9A01A with LVGL

[`ROUND-GC9A01A-LVGL.md`](ROUND-GC9A01A-LVGL.md) is a starting config for a 240x240 round panel with an LVGL arc and CST816 touch, plus the round-specific gotchas (invisible corners, 25% buffer, rotation under `lvgl:`). Not tested on hardware yet.

[`HOST-COMPAT-MATRIX.md`](HOST-COMPAT-MATRIX.md) is how I read Waveshare-style host×panel wikis and BuyDisplay per-MCU tutorial tables when we draft `/compatible/*` pages (Pi cable generations + dtoverlay letters, ESP32-P4 `esp_lcd_*` / BSP brightness names, Arduino SPI FPS framing). Includes P4 IDF clock and 3.5″ lookalike pitfalls we already hit in public threads — **not bench-tested pinouts**.

## ST77922 / Sitronix silent black

[`ST77922-SLPOUT-BLACK.md`](ST77922-SLPOUT-BLACK.md) covers the "backlight on, logs clean, panel black" trap on ST77922 (and similar Sitronix parts): `mipi_spi` still appends only **10 ms** after `SLPOUT` before `DISPON`, while the silicon wants ~120 ms. Workaround is your own `[0x11]` + `delay 200ms` at the end of `init_sequence` ([esphome#19010](https://github.com/esphome/esphome/issues/19010)). **Not bench-tested here.**

## JD9365 / Waveshare 10.1 DSI panel enable

[`JD9365-PANEL-ENABLE-I2C.md`](JD9365-PANEL-ENABLE-I2C.md) covers Waveshare 10.1-DSI-TOUCH-A on ESP32-P4: panel enable is I²C `0x45` / reg `0x95` (JD9365 vendor path), not a GPIO — without it `mipi_dsi` wedges after ~19 DCS cmds into watchdog/safe mode ([esphome#21451](https://github.com/esphome/esphome/issues/21451)). **Not bench-tested here.**

## JD9365 runtime backlight stuck (PWM path)

[`JD9365-BACKLIGHT-STUCK.md`](JD9365-BACKLIGHT-STUCK.md) covers the other half of that split: sysfs/PWM brightness writes succeed but the physical backlight freezes (often after a trip through 0) until full reboot — cite [techo5#253](https://github.com/HuskerMinion/techo5/issues/253) (Athena JD9365 KD-HSD `0x31`). Panel-enable I²C ≠ backlight PWM. **Not bench-tested here.**

## Panel letter / SKU trap (Waveshare DSI)

[`PANEL-LETTER-SKU-TRAP.md`](PANEL-LETTER-SKU-TRAP.md) covers why Waveshare `4-DSI-TOUCH-C` / `10.1-DSI-TOUCH-A` letters are part of the SKU: dtoverlay params (`4_0_inch_c` vs `10_1_inch_a`), different `esp_lcd_jd9365` vs `esp_lcd_jd9365_10_1` packages, and I²C backlight/enable regs on `0x45`. Cite Waveshare wikis + [esphome#21451](https://github.com/esphome/esphome/issues/21451). **Not bench-tested here.**

## ESP32-P4 MIPI DSI bring-up

[`ESP32-P4-MIPI-DSI-BRINGUP.md`](ESP32-P4-MIPI-DSI-BRINGUP.md) collects the public P4 + MIPI DSI traps in one place: don’t hard-code `MIPI_DSI_PHY_CLK_SRC_DEFAULT` on P4 v3.x + IDF 6.1 (abort in `esp_lcd_new_dsi_bus`; [Waveshare ESP32-display-support#43](https://github.com/waveshareteam/ESP32-display-support/issues/43), [iclforge#1604](https://github.com/iainchesworthlabs/iclforge/pull/1604)), panel-letter SKU trap, JD9365 I²C enable `0x45`/`0x95`, long init >127B, and host-only 2-lane boards like LILYGO T-Halow P4. Pointers into the other notes above. **Not bench-tested here.**

## Long CUSTOM init (>127 data bytes)

[`LONG-INIT-127.md`](LONG-INIT-127.md) covers ESPHome's historical `x & 0x7F` length mask on `mipi_spi` / `mipi_rgb` / `mipi_dsi` / `epaper_spi` (and friends): CUSTOM commands with more than 127 data bytes misread as Malformed or get truncated. Cite [esphome#21440](https://github.com/esphome/esphome/pull/21440) / [#21441](https://github.com/esphome/esphome/pull/21441) (closed unmerged); workaround is split long dumps when the datasheet allows. **Not bench-tested here.**

## ST7789 + XPT2046 shared CS / wrong Marauder bin

[`XPT2046-SHARED-CS.md`](XPT2046-SHARED-CS.md) covers the "panel paints, resistive touch dead" trap on SPI TFT + XPT2046 boards: `…ili9341…` Marauder bins on an ST7789 glass, and TFT CS tied to touch CS. Cites [ESP-Web-Flasher#2](https://github.com/Aryann019x/ESP-Web-Flasher/issues/2) (Bruce ESP32-S3 + ST7789 2.8″, still open when written). **Not bench-tested here.**

## Need the datasheet or a different panel?

Datasheets, pin-outs and init sequences can be requested on each product page or via
https://www.displayengineer.com/ask-an-engineer.

Issues and pull requests with tested results (board, ESPHome version, `data_rate`, photo) are welcome.

## License

[MIT](LICENSE)
