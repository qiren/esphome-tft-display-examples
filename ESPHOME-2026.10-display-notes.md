# ESPHome 2026.10.0: what changed for TFT / RGB displays

ESPHome 2026.10.0 came out on 2026-10-21. Most of the release is about ESP8266 builds and compressed OTA, but a few display fixes landed too, and two of them explain problems people have been asking about on the forums for months. This note only lists what the changelog and the merged PRs actually say. We have not re-tested these on hardware yet.

## ST7701S panel black after a reboot (mipi_rgb)

If you have a 480x480 ST7701S board (Waveshare ESP32-S3-Touch-LCD-4, 2.1, Guition 4848S040 and similar) on `mipi_rgb` and the screen comes up fine right after flashing but stays black after a plain reboot, this is the one to look at. The report is [esphome#17636](https://github.com/esphome/esphome/issues/17636) ("display black after any reboot without reflash").

The fix is [esphome#17646](https://github.com/esphome/esphome/pull/17646). In the author's words, the ST7701S was getting a duplicate SWRESET with no delay after it, and "the published delay (5ms) after reset for an ST7701S appears to be insufficient". The PR removes the duplicate and makes the ST7701S model wait 50 ms after reset (other models keep the 10 ms default). It also moves the init sequence so it is only sent after the rest of setup has succeeded.

`reset_delay` is a model default inside the component, not a YAML option you can set. Updating to 2026.10.0 alone is usually not enough: you still need a real `reset_pin` that actually toggles the panel RESX line, then a power cycle. On a lot of these Waveshare / Guition boards that pin sits behind an I2C expander, and the model preset has to name it (or you set it in YAML). The reporter on [esphome#17636](https://github.com/esphome/esphome/issues/17636) tried the #17646 build first and still got black-after-reboot until LCD_RST was driven through the expander; after that, cold boots survived. We have not run that board ourselves, this is just what they wrote in the issue thread.

If your own firmware (ESP-IDF, Arduino) drives an ST7701S directly, give it the same 50 ms after reset rather than the published 5 ms, and make sure RESX is actually pulsed, not left floating on an expander.

Backlight on with a black screen can still have other causes on these boards: check that the expander shows up in the boot log I2C scan before blaming the panel.

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

## On dev after 2026.10.0, not released yet

If your LilyGo T-Display S3 Pro feels sluggish on the `T-DISPLAY-S3-PRO` preset, that's not your imagination. I checked the 2026.10.0 model file and it doesn't set `data_rate`, so the ST7796 runs at the `mipi_spi` default of 10MHz. [esphome#21370](https://github.com/esphome/esphome/pull/21370) was merged into dev on Oct 22 and sets the preset to 40MHz. The PR says a full 480x222 frame took about 170 ms at 10MHz. The math agrees: 480 x 222 x 16 bits is about 1.7 Mbit, so roughly 170 ms at 10MHz and about 43 ms at 40MHz, before LVGL adds anything. Until it lands in a release you can set it yourself, since a config value still wins over the preset:

```yaml
display:
  - platform: mipi_spi
    model: T-DISPLAY-S3-PRO
    data_rate: 40MHz
```

The PR author tested it with an SD card sharing the same SPI bus. I find the width x height x 16 / clock trick handy for any SPI panel. It tells you the best case for a full redraw before you start blaming LVGL.

Round 412x412 SPD2010 folks, one to watch: [esphome#21354](https://github.com/esphome/esphome/pull/21354) (still open when I wrote this) moves the Waveshare 1.46" settings into a generic `SPD2010` model and adds `SEEED-SENSECAP-WATCHER`. The detail I liked is that on the Watcher the panel reset is tied to the chip reset, so there's no reset GPIO and the model falls back to a software reset. If it gets merged, a custom SPD2010 board could start from `model: SPD2010` plus its own pins. I haven't run either board myself.

## "Pin 2 is used in multiple places" on the Waveshare 7" after updating

This one showed up the day after the release ([esphome#21379](https://github.com/esphome/esphome/issues/21379)). Someone's ESP32-S3-Touch-LCD-7 config compiled fine on 2026.9.1, and on 2026.10.0 `esphome config` stops with `Pin 2 is used in multiple places`, pointing at both a `switch: platform: gpio` and the `mipi_rgb` display. The switch was their backlight, on CH422G pin 2.

I went and read the model file. The `ESP32-S3-TOUCH-LCD-7-800X480` preset already claims that pin: it sets `enable_pin` to CH422G pins 2 and 6 (the 4.3" preset it extends uses pin 2 as enable and pin 3 as reset). So the display was already switching pin 2 on at boot, and the extra switch was a second owner of the same pin. That part didn't change between versions, the preset is identical in 2026.9.1 and 2026.10.0.

What changed is the check. [esphome#18939](https://github.com/esphome/esphome/pull/18939), which landed in 2026.10.0, makes the pin registry group expander pins by the hub they resolve to. As far as I can read `esphome/pins.py`, before that the preset's pins (hub id left empty) and your pins (`ch422g: ch422g_hub`) were counted in different buckets, so the clash slipped through. Now they're counted as the same CH422G and the duplicate gets caught. I'd call that the check finally doing its job rather than a regression.

Two ways out. If you never turn the backlight off, just delete the switch, the preset already enables it. If you do want a backlight switch in HA, take pin 2 out of the display's `enable_pin` and keep your switch:

```yaml
display:
  - platform: mipi_rgb
    model: ESP32-S3-TOUCH-LCD-7-800X480
    enable_pin:
      - ch422g: ch422g_hub
        number: 6
    # rest of your display config unchanged
```

I kept pin 6 because the preset sets it and I don't know what it gates on every board revision. The maintainer's first reply on the issue was to double check which CH422G pin the backlight is really on, so look at your board's schematic before you copy pin numbers from anyone, me included. And the 7C is a different story: it uses a CH32V003 expander on other I2C pins, and there's no ESPHome preset for it yet, so people are starting from `ESP32-8048S070` and overriding the pins. I haven't run either board.

## Still true: ili9xxx / st7789v / st7735 go away in 2027.5.0

The 2026.10.0 release notes do not mention it, but the removal of `ili9xxx`, `st7789v` and `st7735` (PRs #20687 to #20690) is still scheduled for 2027.5.0. See [`MIGRATING-to-mipi_spi.md`](MIGRATING-to-mipi_spi.md).

Sources: [ESPHome 2026.10.0 changelog](https://esphome.io/changelog/2026.10.0/), the 2026.10.0 [`lilygo.py`](https://github.com/esphome/esphome/blob/2026.10.0/esphome/components/mipi_spi/models/lilygo.py) and [`mipi_rgb/models/waveshare.py`](https://github.com/esphome/esphome/blob/2026.10.0/esphome/components/mipi_rgb/models/waveshare.py) model files, [`esphome/pins.py`](https://github.com/esphome/esphome/blob/2026.10.0/esphome/pins.py), and the PRs linked above.

---
Written by Display Engineer (TFT/LCD module maker, https://www.displayengineer.com). Corrections with test results are welcome as issues or PRs.
