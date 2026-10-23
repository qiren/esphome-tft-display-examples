# Waveshare panel letter ≠ “any same-size DSI”

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). This note is about a buyer trap I keep seeing in public threads: someone copies a `dtoverlay=…` or `esp_lcd_*` dependency from a similarly sized Waveshare wiki page, and the backlight comes up but the panel stays black. **Not bench-tested here** — I’m only repeating what the public wikis and one ESPHome issue say.

## The letter is part of the SKU

Waveshare’s DSI touch line names panels like `4-DSI-TOUCH-C` and `10.1-DSI-TOUCH-A`. The trailing letter is not a cosmetic revision sticker. It picks the driver IC, the Pi overlay parameter, and often a different Espressif component name.

On the [4-DSI-TOUCH-C wiki](https://www.waveshare.com/wiki/4-DSI-TOUCH-C) FAQ they state the display IC is **JD9365** and touch is **GT928**. The Pi snippet they want you to paste is:

```text
dtoverlay=vc4-kms-v3d
dtoverlay=vc4-kms-dsi-waveshare-panel-v2,4_0_inch_c
```

That `4_0_inch_c` string is the letter. Flash `4_0_inch_a` (or whatever you found on a different page) onto a **-C** panel and you get the classic backlight-and-black.

Same pattern on the [10.1-DSI-TOUCH-A wiki](https://www.waveshare.com/wiki/10.1-DSI-TOUCH-A). They put a hard warning before the config block: applicable model is **10.1-DSI-TOUCH-A**, *not* “10.1inch DSI LCD (C)” or other 10.1″ DSI SKUs. Wrong model → “screen not lighting up, black screen, or backlight on but no image.” Their overlay parameter is `10_1_inch_a`, and the FAQ lists display IC **JD9365** / touch **GT9271**.

So even when two letters both use a JD9365-class panel, the wiki still refuses cross-SKU copy-paste. Treat the letter as part of the part number.

## ESP32-P4: the component name follows the SKU too

Same two wiki pages, ESP32-P4 section:

- 4-DSI-TOUCH-C → `idf.py add-dependency "waveshare/esp_lcd_jd9365"`
- 10.1-DSI-TOUCH-A → `idf.py add-dependency "waveshare/esp_lcd_jd9365_10_1"`

Those are different registry packages. Grabbing the 4″ dependency for a 10.1″ board (or the other way around) is the same class of mistake as swapping the dtoverlay letter.

Backlight / panel-power on these boards is also not “just a GPIO.” The 4″ C wiki documents writing `0x00`–`0xFF` to I²C device **0x45**, register **0x96**. The 10.1″ A wiki uses the same `0x45` device but register **0x86** for backlight. And for ESPHome bring-up of the 10.1″ A, [esphome#21451](https://github.com/esphome/esphome/issues/21451) shows panel *enable* is another I²C write on `0x45` / reg `0x95` (`0x11` then `0x17`) — without it `mipi_dsi` wedges after ~19 DCS commands into watchdog / safe mode. We already expanded that path in [`JD9365-PANEL-ENABLE-I2C.md`](JD9365-PANEL-ENABLE-I2C.md).

## What I actually check before pasting someone else’s config

Read the exact wiki URL that matches the silkscreen / product title, including the letter. Copy the overlay parameter and `esp_lcd_*` name from *that* page only. If the FAQ lists a different display IC or touch IC than the thread you found, stop — you’re on the wrong SKU. On ESP32-P4, confirm whether panel power / backlight is an I²C register on `0x45` before you burn a day chasing DSI lane rates.

## Disclosure

I work for Display Engineer. One link above points at our site; the Waveshare / ESPHome URLs are the primary sources. None of the overlays or I²C sequences in this file were measured on our bench for this commit.
