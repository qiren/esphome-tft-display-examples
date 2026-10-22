# White screen with the backlight on? Check TFT_RST before anything else

I keep seeing the same pattern. Someone wires a cheap SPI TFT to an ESP32, copies a TFT_eSPI `User_Setup`, flashes, and gets a bright white (or washed blank) panel. SPI pins look right. Backlight is on. Model define is correct. Still nothing useful on the glass.

On a lot of modules the FPC `RESET` pin is already tied to the MCU's `EN` / `RST` line on the breakout, or the seller tells you to jumper TFT RST to the board RST so one reset clears both chips. That wiring is fine. What breaks is leaving `TFT_RST` pointed at a GPIO that isn't actually connected to the panel.

Bodmer's own `User_Setup.h` says this explicitly: set `TFT_RST` to `-1` if the display RESET is connected to the ESP / NodeMCU RST (or to 3.3V). The comments are right next to the pin defines — see the raw file on
[Bodmer/TFT_eSPI `User_Setup.h`](https://raw.githubusercontent.com/Bodmer/TFT_eSPI/master/User_Setup.h)
(search for `Set TFT_RST to -1`). When you leave a random pin number in there, the library still toggles that GPIO at boot. The panel never sees a real reset pulse on its RST pad, the init sequence goes into a chip that is still half-awake, and you get the classic white screen with backlight on.

What I'd do:

```cpp
#define TFT_RST  -1   // panel RST tied to EN / board RST
```

Then rebuild. If the glass comes alive, that was it. If it stays white, go back to DC / CS / MOSI wiring and the model define — those are the next usual suspects, and we already cover them in the
[white screen guide](https://www.displayengineer.com/guides/tft-display-white-screen-causes-fixes).

Same idea in ESPHome `mipi_spi`: if the module's RST is hard-wired to EN, don't invent a `reset_pin` on a GPIO that isn't routed. Leave `reset_pin` out and let the shared EN do the power-on reset. We ran into the opposite case (missing `reset_pin` when the expander *does* need a dedicated pulse) on Waveshare RGB boards after reboot — that's a different note in `ESPHOME-2026.10-display-notes.md`. Here the bug is the other way around: a software reset pin that doesn't exist on the wire.

Not bench-tested on every module. The `-1` rule is copied from Bodmer's setup comments and from how these FPC breakouts are usually wired, not from a logic-analyser capture of our own. This repo is maintained by [Display Engineer](https://www.displayengineer.com/), a display module maker.
