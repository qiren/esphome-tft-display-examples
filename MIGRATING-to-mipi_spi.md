# Moving ESPHome displays from ili9xxx / st7789v / st7735 to mipi_spi

We make small TFT modules, and lately the question we get most from ESPHome users is "my config now prints a deprecation warning, what do I actually change?" So we sat down with the ESPHome docs and the `mipi_spi` source and wrote down the before/after for the controllers people ask about most. Pins in every snippet are just examples.

## When the old platforms go away

`ili9xxx`, `st7789v`, `st7735` and `qspi_dbi` have been deprecated since April 2026, but until recently nobody had put a date on it. That changed with four PRs merged into ESPHome's `dev` branch on 2026-10-14: [#20687](https://github.com/esphome/esphome/pull/20687) (ili9xxx), [#20688](https://github.com/esphome/esphome/pull/20688) (st7735), [#20689](https://github.com/esphome/esphome/pull/20689) (st7789v) and [#20690](https://github.com/esphome/esphome/pull/20690) (qspi_dbi). Each one sets removal for ESPHome 2027.5.0, which the PR text describes as "six months after this ships in 2026.11", and the compile-time warning now names `mipi_spi` as the replacement.

Nothing breaks today. Once 2027.5.0 lands, though, a YAML that still says `platform: st7789v` won't compile, so it's worth doing the move while the old config still works and you can compare the two.

## The stuff that changes for every panel

Your `spi:` block stays the same, and `cs_pin`, `dc_pin` and `reset_pin` keep their names. Most of the surprises are elsewhere.

The default SPI clock is lower. `mipi_spi` defaults `data_rate` to 10MHz. The old `ili9xxx` defaulted to 40MHz and `st7789v` to 20MHz. If the screen suddenly redraws slowly after the switch, that's the reason, so write your old speed in explicitly.

The driver doesn't touch the backlight anymore. The old `st7789v` platform had a `backlight_pin` option. `mipi_spi` leaves the backlight alone, so it needs its own `output` and `light` (or a GPIO switch). Skip this and you can end up with a perfectly initialised panel sitting in the dark.

Rotation is just `rotation:` now. In `ili9xxx`, `rotation` was software only and you used `transform` for hardware rotation. With a built-in `mipi_spi` model the docs say to use `rotation` and the driver turns it into a hardware rotation when the chip supports it. Drop the old `transform:` block unless you're on `CUSTOM`, and keep `dimensions` at the panel's native size instead of swapping width and height yourself.

SPI mode follows the CS pin. `mipi_spi` uses MODE0 when you give it a `cs_pin` and MODE3 for single-bit SPI without one. Some of the cheap 240x240 ST7789 boards have no CS line, so check this if one of those stays blank.

8-bit colour moved too: `eightbitcolor: true` (st7789v) and `eight_bit_color: true` (st7735) both become `color_depth: 8`.

## ILI9341

Before:

```yaml
display:
  - platform: ili9xxx
    model: ILI9341
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    invert_colors: false
    rotation: 90
```

After:

```yaml
display:
  - platform: mipi_spi
    model: ILI9341
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    data_rate: 40MHz        # old ili9xxx default, mipi_spi would use 10MHz
    invert_colors: false    # TESTING-NEEDED: flip if colours look like a negative
    rotation: 90
```

`invert_colors` was required in `ili9xxx`. In `mipi_spi` it's optional and defaults to false. Also worth knowing: plenty of boards sold as "ILI9341" actually carry an ST7789. The 2.8" CYD is the classic case, and `mipi_spi` has three separate board models for it (`ESP32-2432S028`, `ESP32-2432S028-7789`, `ESP32-2432S028-9342`). If your colours look washed out on a dual-USB CYD, the controller is the first thing to check.

## ST7789V, 240x320

Before (the old `st7789v` platform):

```yaml
display:
  - platform: st7789v
    model: Custom
    height: 320
    width: 240
    offset_height: 0
    offset_width: 0
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    backlight_pin: GPIO14
```

After:

```yaml
output:
  - platform: ledc
    pin: GPIO14             # TESTING-NEEDED: polarity depends on your backlight circuit
    id: backlight_pwm

light:
  - platform: monochromatic
    output: backlight_pwm
    name: "Backlight"
    restore_mode: ALWAYS_ON

display:
  - platform: mipi_spi
    model: ST7789V          # built-in model, native 240x320
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    data_rate: 20MHz        # old st7789v default
    invert_colors: false    # TESTING-NEEDED
```

The `ST7789V` model already knows it's 240x320, so you don't need `dimensions` unless the image is shifted. If you were on one of the old presets like `TTGO TDisplay 135x240`, look at the board list in the `mipi_spi` docs first, there's a `T-DISPLAY` model. For 240x240 ST7789 modules, set `dimensions` to 240x240. The two GeekMagic presets in the `mipi_spi` source do exactly that, with zero offsets and `invert_colors: true`, which is a reasonable place to start.

There's a full ST7789V example for our 2.8" module in [`esphome/pj2801h22-st7789v-esp32s3.yaml`](esphome/pj2801h22-st7789v-esp32s3.yaml).

## ST7735, 128x160

Before:

```yaml
display:
  - platform: st7735
    model: INITR_BLACKTAB
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    device_width: 128
    device_height: 160
    col_start: 0
    row_start: 0
    use_bgr: false
    invert_colors: false
```

After:

```yaml
display:
  - platform: mipi_spi
    model: ST7735
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    dimensions:
      width: 128
      height: 160
      offset_width: 0       # was col_start, TESTING-NEEDED
      offset_height: 0      # was row_start, TESTING-NEEDED
    invert_colors: false
```

There's no BLACKTAB / GREENTAB / REDTAB choice anymore, just one `ST7735` model. The tab variants mostly come down to different offsets and RGB/BGR order, so if the picture lands a couple of pixels off, carry your old `col_start` / `row_start` into the offsets. One detail from the source: the `ST7735` model sets `color_order` to rgb itself, even though the general `mipi_spi` default is bgr. That matches the old `use_bgr: false`. If you had `use_bgr: true`, set `color_order: bgr`. The 0.96" 80x160 ones work the same way, just with `dimensions` of 80x160 and whatever offsets you used before.

## GC9A01, 240x240 round

Before:

```yaml
display:
  - platform: ili9xxx
    model: GC9A01A
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    invert_colors: true
```

After:

```yaml
display:
  - platform: mipi_spi
    model: GC9A01A
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    data_rate: 40MHz
    invert_colors: true     # TESTING-NEEDED
```

The model name is `GC9A01A` in both platforms. There's no plain `GC9A01` in the list, so don't drop the A. `invert_colors: true` is a guess for your panel, but it's what the `ESP32-2424S012` round CYD preset in the `mipi_spi` source uses.

## ILI9488, 320x480

Before:

```yaml
display:
  - platform: ili9xxx
    model: ILI9488
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    invert_colors: false
```

After:

```yaml
display:
  - platform: mipi_spi
    model: ILI9488          # ILI9488_A is the same chip with an alternative gamma table
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    data_rate: 40MHz
    invert_colors: false    # TESTING-NEEDED
```

The `ILI9488` model in the `mipi_spi` source already sets `pixel_mode` to 18bit, which is what this chip needs over SPI, so leave `pixel_mode` out. That also means 3 bytes per pixel on the wire. If you're still choosing a 3.5" 320x480 panel and frame rate matters, an ST7796 (`model: ST7796`) runs 16-bit over SPI.

## No preset? ILI9163 and friends

ILI9163 isn't in the `mipi_spi` chip list. We checked the docs (2026.9.1) and the model files on both the release and dev branches on 2026-10-19. That matters to us because our 1.77" 128x160 module uses an ILI9163V. For a chip like that you use `model: CUSTOM`, which needs both `dimensions` and `init_sequence`, since there's no built-in init at all.

```yaml
display:
  - platform: mipi_spi
    model: CUSTOM
    cs_pin: GPIO10
    dc_pin: GPIO9
    reset_pin: GPIO8
    dimensions:
      width: 128
      height: 160
    init_sequence:          # TESTING-NEEDED: paste the panel vendor's init table here
      - [0x11]              # sleep out, if the vendor table starts with it
      - delay 120ms
      - [0x29]              # display on
```

Each line is one command byte followed by its data bytes. If you're copying from Arduino code, watch for tables that store a length byte before the data. The docs say not to copy that, since `mipi_spi` works out the length from the list. If rotation acts weird on a custom panel that can't swap axes, the docs suggest `transform: disabled` so rotation falls back to software.

## If it still looks wrong

| What you see | Try this first |
|---|---|
| Colours look like a photo negative | Flip `invert_colors` |
| Red and blue swapped | `color_order: rgb` (or `bgr`) |
| Mirrored or upside down | `rotation` first. On `CUSTOM`, `transform` with `mirror_x` / `mirror_y` (needs `dimensions`) |
| Picture shifted, junk strip on one edge | `dimensions` with `offset_width` / `offset_height` |
| White screen, backlight on | Check `model` and the DC wire, drop `data_rate` to 10MHz, and on `CUSTOM` make sure the full init table is there |

---

TESTING-NEEDED: none of the YAML above has been run on hardware yet. It's written from the ESPHome docs (2026.9.1) and the `mipi_spi` source, checked on 2026-10-19. This guide comes from DisplayEngineer, a display module maker. Our ST7789V notes are at https://www.displayengineer.com/driver-ic/st7789v
