# GC9A01A round 240x240 with LVGL in ESPHome (mipi_spi)

If you just got a 1.28" round panel, the first screen you probably want is a ring you can drag with a number in the middle. Getting there in ESPHome is mostly easy, but three small things catch people: the corners you can't see, the LVGL buffer size, and where `rotation` goes once LVGL is involved. So here's the config I'd start from, with notes on each of those.

I haven't run this exact file on hardware yet. It's written from the ESPHome docs and the 2026.10.0 source, so anything marked `# TESTING-NEEDED` is a starting value, not a measured one. If it works (or doesn't) on your board, an issue with your ESPHome version and a photo helps a lot.

## The config

ESP32-S3 here, but nothing in it is S3-only apart from the pins. Change the GPIOs to match your wiring.

```yaml
esphome:
  name: round-gc9a01a

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf

logger:

spi:
  clk_pin: GPIO12
  mosi_pin: GPIO11

display:
  - platform: mipi_spi
    id: round_display
    model: GC9A01A
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    data_rate: 40MHz           # TESTING-NEEDED, drop to 10MHz on long jumper wires
    invert_colors: true        # TESTING-NEEDED
    auto_clear_enabled: false  # LVGL draws the whole screen itself
    update_interval: never

output:
  - platform: ledc
    pin: GPIO14
    id: backlight_pwm

light:
  - platform: monochromatic
    output: backlight_pwm
    name: "Backlight"
    restore_mode: ALWAYS_ON

lvgl:
  buffer_size: 25%
  widgets:
    - arc:
        id: dial
        align: CENTER
        width: 200
        height: 200
        min_value: 0
        max_value: 100
        value: 40
        adjustable: true
        arc_width: 16
        indicator:
          arc_width: 16
          arc_color: 0x00A0FF
        on_release:
          - lvgl.label.update:
              id: dial_value
              text:
                format: "%.0f"
                args: [ 'x' ]
    - label:
        id: dial_value
        align: CENTER
        text: "40"
```

A couple of things in there that aren't obvious.

`auto_clear_enabled: false` and `update_interval: never` on the display are what the ESPHome LVGL page asks for. Leave them out and the display component and LVGL both try to own the screen.

The model name is `GC9A01A`, with the A. There's no plain `GC9A01` in the `mipi_spi` list. And `invert_colors: true` is a guess for your panel, but it's what the Sunton `ESP32-2424S012` round board preset uses in the `mipi_spi` source (`models/cyd.py`). If your colours look like a photo negative, flip it.

I used `on_release` to update the label because the arc docs warn that `on_value` fires the whole time you're dragging, which costs frame rate. If you want the number to follow your finger, swap it to `on_value` and see if it stays smooth on your board.

## The corners don't exist

The controller has a full 240x240 square of memory, but the glass only shows the circle inside it. Anything you align to `TOP_LEFT` or `BOTTOM_RIGHT` is simply gone. A quick sanity number: the biggest square that fits inside a 240 px circle is about 170x170 (240 divided by the square root of 2). I'd keep text inside that box and leave the edge for rings and arcs.

The arc widget suits this nicely. By default its background runs from 135° to 45°, and 0° is at 3 o'clock going clockwise, so you get a 270° ring with a gap at the bottom. If you want the gap somewhere else, change `rotation` on the arc, not the angles.

## Buffer size

LVGL allocates its own draw buffer, so the `mipi_spi` display doesn't need one when LVGL is in charge. At RGB565 a full 240x240 frame is 240 × 240 × 2 = 115,200 bytes. The LVGL docs say the default is 100% (falling back to 12% at runtime if that allocation fails) and recommend 25% on boards without PSRAM, which here is 28,800 bytes. That's why the config says `buffer_size: 25%`. With PSRAM you can try 100% and see if it feels better.

## Rotation goes on LVGL, not the display

If you need the screen turned, put `rotation: 90` (or 180, 270) under `lvgl:` and leave `rotation` off the display. The ESPHome docs say LVGL then rotates the touch input too, and uses the driver's hardware rotation when it can (most `mipi_spi` models can).

One thing I noticed in the source: the `GC9A01A` chip definition in `esphome/components/mipi_spi/models/ili.py` (2026.10.0) sets `mirror_x=True` by default. So if your text comes out mirrored, that's a scan direction thing, not your wiring. Fix it with `transform:` (`swap_xy`, `mirror_x`, `mirror_y`), which needs `dimensions:` set as well.

## Adding touch (CST816)

A lot of round modules with touch use a CST816-family controller on I2C. ESPHome has a `cst816` touchscreen platform for it:

```yaml
i2c:
  sda: GPIO6
  scl: GPIO7

touchscreen:
  - platform: cst816
    id: round_touch
    interrupt_pin: GPIO5
    reset_pin: GPIO4
    # skip_probe: true   # some CST816S only answer I2C after the first touch
```

With a single touchscreen, LVGL picks it up on its own, no `touchscreens:` entry needed. The `skip_probe` comment is straight from the ESPHome docs: some CST816S chips don't reply to the ID read until you've touched them once, so if the log says the touch controller wasn't found, try that first.

## If you have the Sunton round board

The `ESP32-2424S012` (1.28" round, GC9A01A) is a board preset in `mipi_spi`, so `model: ESP32-2424S012` sets the pins and inversion for you and you can skip most of the display block above.

## Sources

- ESPHome `mipi_spi` display: https://esphome.io/components/display/mipi_spi/
- ESPHome LVGL (buffer size, rotation, `auto_clear_enabled`): https://esphome.io/components/lvgl/
- ESPHome LVGL widgets (arc angles, `on_value` vs `on_release`, label text format): https://esphome.io/components/lvgl/widgets/
- ESPHome `cst816` touchscreen: https://esphome.io/components/touchscreen/cst816/
- `mipi_spi` source at 2026.10.0, `models/ili.py` (GC9A01A) and `models/cyd.py` (ESP32-2424S012): https://github.com/esphome/esphome/tree/2026.10.0/esphome/components/mipi_spi/models

I work at a display maker, so take that into account. We make round GC9A01 modules among other things, and the controller notes are on our [GC9A01 page](https://www.displayengineer.com/driver-ic/gc9a01). The config itself is plain ESPHome and should behave the same on any GC9A01A 240x240 panel.
