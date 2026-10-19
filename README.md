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

## Need the datasheet or a different panel?

Datasheets, pin-outs and init sequences can be requested on each product page or via
https://www.displayengineer.com/ask-an-engineer.

Issues and pull requests with tested results (board, ESPHome version, `data_rate`, photo) are welcome.

## License

[MIT](LICENSE)
