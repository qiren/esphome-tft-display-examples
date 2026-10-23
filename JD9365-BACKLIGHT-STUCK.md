# JD9365: sysfs backlight writes, but the panel stays stuck bright

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). This is **not bench-tested here**. I'm writing it down because [HuskerMinion/techo5#253](https://github.com/HuskerMinion/techo5/issues/253) (Athena / JD9365 KD-HSD, panel id `0x31`, still open when I wrote this) is a clean example of a split people keep missing: the **panel-enable** path and the **backlight** path are not the same wire, and a successful sysfs write does not mean the light followed you.

If you landed here chasing a black Waveshare 10.1" DSI on ESP32-P4, start with [`JD9365-PANEL-ENABLE-I2C.md`](JD9365-PANEL-ENABLE-I2C.md) instead. That note is about I²C `0x45` / reg `0x95` never getting written, so DSI wedges and you end up in safe mode. This note is the other half: the panel is already painting, touch and audio are fine, and the *brightness control* is what died.

## What #253 looks like

On Athena (Echo Show 8 2nd Gen, TECHO5 Linux) with the JD9365 KD/HSD glass:

- Fresh boot: auto and manual brightness work.
- After some uptime (sometimes hours, sometimes after a daylight jump), the physical backlight sticks high.
- `/sys/class/backlight/backlight-lcd0/brightness` still accepts writes. Numbers move. `actual_brightness` moves with them.
- The room light sensor still reads sensible values.
- Restarting the TECHO5 / Home Assistant app does **not** fix it.
- A full Linux `reboot` (or power cycle) does.

So userspace and the backlight class both think they are talking. The panel image path is fine. Only the photons stopped listening.

## Panel enable vs backlight — two different jobs

On a lot of JD9365 / Waveshare-style carriers you have at least two independent controls:

1. **Panel enable / power gate** — often an I²C expander or companion chip (Waveshare's public `esp_lcd_jd9365` path uses device `0x45`, reg `0x95` to bring the glass up before any DSI traffic; see the sibling note and [esphome/esphome#21451](https://github.com/esphome/esphome/issues/21451)). Miss that and you get black screen / FIFO wedge / safe mode. You do **not** get a stuck-bright picture with working touch.

2. **Backlight drive** — PWM (or an I²C brightness reg on the same companion, e.g. `0x96` on that Waveshare path) feeding an LED driver. On the Athena unit in #253 the maintainer was clear: the backlight runs from a **PWM on the main SoC**, not from a register inside the JD9365 itself. `actual_brightness` in sysfs is "last value written", not a read-back from the LED string.

Those two can fail independently. A working I²C enable does not prove the PWM path is healthy, and a writable sysfs node does not prove the LED driver is still tracking duty cycle.

## Why sysfs can succeed while the light is stuck

In #253 the debugfs PWM dump showed the kernel really changing duty (25% / 50% / 75% / 100% all updated). The screen stayed fully bright anyway. So the fault was **past** the PWM block — the pinmux still pointed at the PWM, the clock looked alive, and still the LED string did not follow.

A second pattern from the same thread: after someone forced `echo 0` into the backlight sysfs (screen goes dark), then `echo 300` (screen comes back on), mid-level writes stopped having any visible effect. One reporter could recover by going `0 → 1000` (full) as a blink. Another saw the same "stops following after a trip through 0" on the stock kernel. The maintainer's read was that at brightness 0 the driver was switching the PWM **off** entirely instead of holding 0% duty, and after that off→on edge the hardware stopped tracking new settings until a full reboot re-initialized it.

I have not reproduced that on our modules. Treat it as a public hypothesis from the issue, not a datasheet claim. The useful takeaway for anyone wiring JD9365 (or any MIPI panel with a separate BL rail) is: **a successful write and a moving `actual_brightness` only prove the software side of the backlight class**. They do not prove the LED driver, the enable FET, or the PWM-to-driver handoff still work.

## What I'd check before assuming the panel IC is bad

When brightness freezes but the image keeps updating:

- Confirm which path owns brightness on *your* board. SoC PWM GPIO? I²C reg on a companion (`0x45`/`0x96` style)? A dedicated LED-driver EN/PWM pin? Do not debug the JD9365 DCS table for a backlight that never touches the panel bus.
- If you have debugfs: dump the PWM requested vs actual config while you move the slider. If duty moves and light does not, look past the PWM (driver chip, enable rail, cabling). If duty does not move, look at the backlight driver / pinmux first.
- Note whether the freeze only appears after a trip through **0** / screen-off. App restart vs full power-on reset matter — #253 only recovered on Linux reboot, not on an HA restart. That is a strong hint the hang lives below userspace.
- On ESPHome / Waveshare P4 DSI bring-ups, keep the I²C panel-enable (`0x95`) and the brightness reg (`0x96` or GPIO/`ledc`) mentally separate. Safe-mode loops from a missing enable look nothing like a stuck-bright backlight; mixing the two wastes a day.
- Do not invent a "JD9365 backlight register" fix from this Athena report alone. That board's BL is SoC PWM. Your carrier may differ — check the schematic / Waveshare wiki for the exact SKU letter.

## Sources

- [HuskerMinion/techo5#253](https://github.com/HuskerMinion/techo5/issues/253) — Athena JD9365 KD-HSD `0x31`, sysfs writable / physical BL stuck, reboot-only recovery (open as of 2026-10-23).
- Sibling note in this repo: [`JD9365-PANEL-ENABLE-I2C.md`](JD9365-PANEL-ENABLE-I2C.md) (I²C enable, not backlight).
- Public ESPHome/Waveshare context for the enable half: [esphome/esphome#21451](https://github.com/esphome/esphome/issues/21451).

No MOQ, price, or lead time in this note — if you need a module or a datasheet, use the ask-an-engineer form on the site rather than guessing from a forum thread.
