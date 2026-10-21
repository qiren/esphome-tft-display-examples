# ESPHome 2026.10.0: what changed for TFT / RGB displays

ESPHome 2026.10.0 came out on 2026-10-21. Most of the release is about ESP8266 builds and compressed OTA, but a few display fixes landed too, and two of them explain problems people have been asking about on the forums for months. This note only lists what the changelog and the merged PRs actually say. We have not re-tested these on hardware yet.

## ST7701S panel black after a reboot (mipi_rgb)

If you have a 480x480 ST7701S board (Waveshare ESP32-S3-Touch-LCD-4, 2.1, Guition 4848S040 and similar) on `mipi_rgb` and the screen comes up fine right after flashing but stays black after a plain reboot, this is the one to look at. The report is [esphome#17636](https://github.com/esphome/esphome/issues/17636) ("display black after any reboot without reflash").

The fix is [esphome#17646](https://github.com/esphome/esphome/pull/17646). In the author's words, the ST7701S was getting a duplicate SWRESET with no delay after it, and "the published delay (5ms) after reset for an ST7701S appears to be insufficient". The PR removes the duplicate and makes the ST7701S model wait 50 ms after reset (other models keep the 10 ms default). It also moves the init sequence so it is only sent after the rest of setup has succeeded.

`reset_delay` is a model default inside the component, not a YAML option you can set, so the practical step is just: update to 2026.10.0 and power-cycle. If your own firmware (ESP-IDF, Arduino) drives an ST7701S directly, it is worth giving it the same 50 ms after reset rather than the published 5 ms.

Backlight on with a black screen can still have other causes on these boards: the reset and CS lines often sit on an I2C expander (PCA9554/TCA9554), so check that the expander shows up in the boot log I2C scan before blaming the panel.

## Still on the old `platform: st7701s`? It never pulses reset_pin

A lot of ST7701S configs passed around the forums (for example the Waveshare ESP32-S3-Touch-LCD-2.1 ones) use the older `st7701s` display platform, not `mipi_rgb`. Reading the 2026.10.0 source of that component (`esphome/components/st7701s/st7701s.cpp`), `reset_pin` is accepted in YAML and printed in the config dump, but nothing in `setup()` ever writes to it. The component sends the init sequence over SPI, then creates the RGB panel and calls `esp_lcd_panel_reset()`, which resets the RGB peripheral, not the panel's RESX line.

So on boards where the panel reset sits on an I2C expander, the line just stays wherever the expander leaves it at boot. If your board pulls it high, everything works and you never notice. If it does not, you get backlight and a black screen whatever you draw. `mipi_rgb` does drive it: it pulses the reset pin high, low, high before init (`mipi_rgb.cpp`), and on 2026.10.0 the ST7701S model also waits 50 ms afterwards (see above).

If you are stuck on a black ST7701S panel with the old platform, moving the same pins to `mipi_rgb` with `model: ST7701S` on 2026.10.0 is the first thing we would try. We have not tested that on a 2.1C board ourselves.

## `rotation: 180` ignored on pure RGB panels (mipi_rgb)

On 2026.9.x, `rotation: 180` was silently ignored on `mipi_rgb` models that have no SPI init interface, and the log printed `Rotation: 0 degrees`. [esphome#20757](https://github.com/esphome/esphome/pull/20757) fixes it by rotating these panels in software. The PR lists the affected models: ESP32-S3-TOUCH-LCD-4.3, WAVESHARE-5-1024X600, ESP32-S3-TOUCH-LCD-7-800X480, ESP32-8048S070, ESP32-8048S050, CROWPANEL-ADVANCE-7 and RPI. ST7701S boards and `CUSTOM` are unchanged.

One side effect to watch for after updating: on those same models a `transform:` block with `mirror_x` or `mirror_y` set to `true` is now rejected with `'mirror_x' is not supported by this model`. Remove the transform and use `rotation:` instead.

## Smaller items

- `mipi_spi` gained a light platform for AMOLED brightness ([esphome#20212](https://github.com/esphome/esphome/pull/20212)), since AMOLED panels have no normal backlight pin.
- `mipi_spi` has a new model for the Waveshare ESP32-S3-Touch-LCD-1.46 (412x412 SPD2010, QSPI) ([esphome#19056](https://github.com/esphome/esphome/pull/19056)). The PR notes that the board's PCA9554 reset pin has to be given in YAML.
- `mipi_spi`, `mipi_rgb` and `mipi_dsi` now keep init sequences in flash instead of RAM.

## Still true: ili9xxx / st7789v / st7735 go away in 2027.5.0

The 2026.10.0 release notes do not mention it, but the removal of `ili9xxx`, `st7789v` and `st7735` (PRs #20687 to #20690) is still scheduled for 2027.5.0. See [`MIGRATING-to-mipi_spi.md`](MIGRATING-to-mipi_spi.md).

Sources: [ESPHome 2026.10.0 changelog](https://esphome.io/changelog/2026.10.0/) and the PRs linked above.

---
Written by Display Engineer (TFT/LCD module maker, https://www.displayengineer.com). Corrections with test results are welcome as issues or PRs.
