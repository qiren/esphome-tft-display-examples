# Waveshare 10.1-DSI-TOUCH-A on ESP32-P4 — panel enable is I²C, not a GPIO

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). This is **not bench-tested here**. I'm writing down what showed up on [esphome/esphome#21451](https://github.com/esphome/esphome/issues/21451) so the next person chasing a black 10.1" DSI panel does not burn a day on lane rate and porch numbers first.

## What it looks like

You flash ESPHome `mipi_dsi` with `model: WAVESHARE-10.1-DSI-TOUCH-A` on an ESP32-P4 carrier (Waveshare ESP32-P4-Module-DEV-KIT style). Boot stops right after `Running Setup`. About 30 s later the task watchdog fires. After ~10 tries you land in safe mode with WiFi/OTA and a blank panel.

With `logger` and `display.mipi_dsi` at `VERY_VERBOSE`, the log always stops after the same count of DCS writes — **19 commands**, last one something like `Command 37`. That number does not move when you stretch reset delay, change lane bit rate, or paste the vendor init table as `CUSTOM`. Writes still return `ESP_OK` because they are only buffering into a FIFO that never drains. A DCS *read* (e.g. panel ID `0x04`) hangs immediately and reports **0** commands sent. That pattern is the giveaway: the panel never came up on the bus.

The crash PC lands in `mipi_dsi_host_ll_gen_is_cmd_fifo_full`, not in your YAML.

## Why

On this Waveshare carrier the panel power enable is **not a GPIO**. Waveshare's `esp_lcd_new_panel_jd9365()` path writes I²C device **`0x45`** (SDA=GPIO07 / SCL=GPIO08 on the kit they use) **before** any DSI traffic:

```c
// vendor shape from the issue report — not copied from our bench
write 0x45 / reg 0x95 <- 0x11
write 0x45 / reg 0x95 <- 0x17
write 0x45 / reg 0x96 <- 0x00
delay 100 ms
write 0x45 / reg 0x96 <- 0xFF
delay 1000 ms
```

`0x95` is the panel enable gate. `0x96` is backlight brightness on the same controller. ESPHome `mipi_dsi` has `enable_pin` for a GPIO enable, and it may drive `0x96` later for brightness, but it **never writes `0x95`**. Panel stays off → DSI host wedges → watchdog → safe mode. Black screen is the symptom, not the root cause.

The sibling **7-DSI-TOUCH-A** (ILI9881C) on a different carrier does *not* need this dance and works with stock `model:` + dimensions. Do not copy the `0x45` workaround onto every Waveshare DSI board by default.

One more trap from the issue: a **soft** reset (OTA, restart button) can look fine if a vendor demo already wrote `0x95` earlier. Only a cold `rst:0x1 (POWERON)` is an honest test.

## Workaround that does not need an ESPHome patch

Replay the vendor enable at `on_boot` priority **900** — after the I²C bus (bus priority 1000) and before display setup — in one synchronous lambda so nothing else races ahead of the panel:

```yaml
esp_ldo:
  - voltage: 2.5V
    channel: 3        # LDO_VO3 -> VDD_MIPI_DPHY on these kits

i2c:
  - id: bus_a
    sda: GPIO07
    scl: GPIO08
    frequency: 400kHz

esphome:
  on_boot:
    - priority: 900
      then:
        - lambda: |-
            auto *bus = id(bus_a);
            uint8_t w[2];
            w[0] = 0x95; w[1] = 0x11; bus->write(0x45, w, 2);
            w[0] = 0x95; w[1] = 0x17; bus->write(0x45, w, 2);
            w[0] = 0x96; w[1] = 0x00; bus->write(0x45, w, 2);
            delay(100);
            w[0] = 0x96; w[1] = 0xFF; bus->write(0x45, w, 2);
            delay(1000);

display:
  - platform: mipi_dsi
    model: WAVESHARE-10.1-DSI-TOUCH-A
    update_interval: never
    auto_clear_enabled: false
    dimensions:
      width: 800
      height: 1280
```

On the reporter's board that jumped from 19 stuck commands to ~202 and `MIPI DSI setup complete`. Pins above match the Waveshare kit in #21451 — check your schematic if the carrier differs. The panel itself has no reset GPIO (`BSP_LCD_RST = NC`); it self-resets on power-up.

Longer post-reset settle, per-command yields, moving `esp_lcd_panel_init()` around, and inlining the 197-command vendor table were all ruled out in the issue — same 19-command wedge every time until `0x95` was written.

## Quick triage

If you see exactly 19 DCS writes then a FIFO busy-wait on a Waveshare P4 + 10.1-DSI-TOUCH-A, try the `0x45` / `0x95` enable before you retune lane rate or porch. If a DCS read hangs at 0 commands, that is the same class of fault. If the 7" ILI9881C sibling works and only the 10.1" JD9365 path fails, that points at the I²C gate too.

Longer term, ESPHome may want a way to declare an I²C panel enable (or document it on the model). Until then the `on_boot` lambda is enough.

## Disclosure

I work for Display Engineer. This note restates public findings from [esphome/esphome#21451](https://github.com/esphome/esphome/issues/21451) (reporter agillis, ESPHome 2026.9.1). We did not run this panel on our bench for this commit. At most one link above points at our site.
