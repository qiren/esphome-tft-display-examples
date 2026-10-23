# Silent black screen on ST77922 / Sitronix — post-SLPOUT delay is too short

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). This is **not bench-tested here**. I'm writing down a pitfall that keeps showing up on public ESPHome issues so we stop treating "black + backlight on + no log error" as a wiring fault first.

## What it looks like

You flash a `mipi_spi` CUSTOM (or early ST77922) config. Logs are clean. `Command 11` and `Command 29` both print. Frames are updating at the expected rate. Backlight is on. Panel stays black.

That pattern matched [esphome/esphome#19010](https://github.com/esphome/esphome/issues/19010) on an LCDwiki ES3C35P (3.5″ 320×480 ST77922 QSPI). Same board worked once the post-sleep delay was long enough.

## Why

`DriverChip.get_sequence()` in `esphome/components/mipi/__init__.py` still appends something like:

```python
sequence.append((SLPOUT,))   # 0x11
sequence.append(delay(10))   # hard-coded 10 ms
sequence.append((DISPON,))   # 0x29
```

Sitronix parts (ST7789 family, ST77922, …) usually want on the order of **120 ms after SLPOUT** before the next command. LCDwiki's own Arduino example for that board does `{0x11, …, delay 120}`. Ten milliseconds is below what the silicon wants, so init "succeeds" and the glass never wakes.

The `delay(0)` *before* SLPOUT is fine — `mipi_spi` resolves it against a 120 ms-after-reset deadline. Only the *post*-SLPOUT wait is the short one. Verified still `delay(10)` on ESPHome `dev` when I checked the file for this note (TESTING-NEEDED if your checkout differs).

User `init_sequence:` is inserted **before** that appended tail. You cannot override the framework's 10 ms from YAML alone, and there is no public `slpout_delay` knob yet (#19010 closed as not_planned with the workaround below).

## Workaround that people actually use

Put your own sleep-out + long delay at the **end** of `init_sequence`. The framework still appends another `0x11` / `0x29`, but by then the panel is already awake so the short gap no longer matters:

```yaml
display:
  - platform: mipi_spi
    model: CUSTOM
    # ... bus / pins / dimensions ...
    init_sequence:
      # ... panel registers from the vendor Arduino sketch ...
      - [0x11]
      - delay 200ms
```

[esphome/esphome#18411](https://github.com/esphome/esphome/pull/18411) (Freenove FNK0104N / ST77922) used a similar long delay at the start of its sequence for the same family. Prefer ending with `0x11` + ≥120–200 ms when you control the CUSTOM sequence yourself.

If a future ESPHome release ships a real ST77922 model with `no_slpout=True` or a longer model-level delay, drop the workaround and use the preset. [#19011](https://github.com/esphome/esphome/pull/19011) was the LCDwiki ES3C35P model PR — check whether it landed in your version before copying CUSTOM YAML forever.

## Quick triage before you rewire the FPC

1. Backlight on, logs happy, panel black → try the `0x11` + `delay 200ms` tail first.
2. Still black → then chase DC/CS/QSPI pins and `bus_mode: quad` (different problem).
3. White screen instead of black → different trap; see [`WHITE-SCREEN-TFT-RST.md`](WHITE-SCREEN-TFT-RST.md) (often `TFT_RST` tied to EN).

## Disclosure

I work for Display Engineer. This file cites public ESPHome issues/PRs and vendor Arduino timing only. No pinouts or FPS numbers were measured on our bench for this commit. At most one link above points at our site.
