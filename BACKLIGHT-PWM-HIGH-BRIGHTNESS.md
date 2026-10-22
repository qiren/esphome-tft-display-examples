# Dimming a 1000-nit backlight from ESPHome

When backlight dimming on a high-brightness panel acts weird, like a faint whine at mid levels or a shimmer near the bottom of the range, the first thing I'd check is the PWM frequency against what the LED driver on the board wants.

A 1000-nit panel isn't a "tie the backlight to 3.3 V through a resistor" part anymore. The LED string sits behind a boost LED driver, and the ESP only feeds that driver's PWM or dim pin. So the two things worth getting right in ESPHome are the PWM frequency and how the brightness range maps onto the driver.

I haven't run these exact files on hardware. They come from the ESPHome docs and the published sheets linked at the bottom, so anything marked `# TESTING-NEEDED` is a starting value, not a measured one.

## How much current we're talking about

Two published examples, both 7" and both rated 1000 cd/m²:

Riverdi's design guide for their 7" HB IPS RGB series says the backlight is a 9x3 LED matrix and needs about 270 mA to reach full brightness. Their reference circuit uses a TPS61169 with a 0.75 ohm feedback resistor, which gives 204 mV / 0.75 ohm, about 272 mA. The datasheet puts typical backlight power at 2592 mW.

Newhaven's NHD-7.0-800480AF-ASXP sheet says 9.3 V / 180 mA for the backlight, luminance 800 min and 1000 typ at 180 mA, and a backlight lifetime of 30,000 hours to half brightness at 25 C.

So plan on a couple of watts just for the backlight, and check your own panel's datasheet for the real numbers. Those two are just to show the size of it.

## PWM frequency

ESPHome's `ledc` output defaults to 1 kHz. Riverdi's guide recommends 5 kHz to 100 kHz for the TPS61169 dim input (200 Hz to 200 kHz for the DIO5661). 1 kHz is under that range for the TPS61169, and it's also right in the band where you can sometimes hear the inductor.

The ESPHome ledc page lists frequencies that keep a clean bit depth. 19531 Hz gives 12 bits (4096 steps), which sits inside the 5 to 100 kHz window and still leaves enough steps for smooth fades near the bottom. On the S3 the LEDC timers are 14-bit max, so going much higher costs you resolution.

```yaml
output:
  - platform: ledc
    id: backlight_pwm
    pin: GPIO2            # whatever drives the LED driver's PWM/EN pin on your board
    frequency: 19531Hz    # 12-bit per the ESPHome ledc table
    min_power: 0.05       # TESTING-NEEDED: some drivers drop out at very low duty
    zero_means_zero: true # so "off" is really off, not 5 %

light:
  - platform: monochromatic
    id: backlight
    name: "Backlight"
    output: backlight_pwm
    restore_mode: ALWAYS_ON
    default_transition_length: 0.5s
```

One gotcha: two adjacent LEDC channels share a timer, so if something else on channel 0/1 needs a different frequency (a buzzer, say), pin the backlight to its own pair with `channel:`.

## Waveshare boards with the CH32V003 expander

On several Waveshare S3 display boards the backlight PWM doesn't come from an ESP32 pin at all, it's the CH32V003 I/O expander's dedicated PWM output. ESPHome has an output platform for it:

```yaml
waveshare_io_ch32v003:
  - id: io_exp
    address: 0x24
    i2c_id: bus_a

output:
  - platform: waveshare_io_ch32v003
    waveshare_io_ch32v003_id: io_exp
    id: bl_pwm
    inverted: true        # the docs example says inverted on the 7B; check yours
    safe_pwm_levels:
      min_value: 1
      max_value: 247

light:
  - platform: monochromatic
    id: backlight
    output: bl_pwm
```

The duty is clamped to 1 to 247 out of 255 by default, so 100 % in Home Assistant isn't quite full duty. That's on purpose in the component. If the screen stays dark at every level, see whether your board also has a separate backlight enable on one of the expander pins (the full example in the docs uses pin 2 through a `power_supply`).

## Heat

At 1000 nits most of those couple of watts turn into heat right behind the glass. If the thing lives outdoors or in a sealed box, I'd dim it with an ambient light sensor rather than running flat out all day. The datasheets I looked at rate the LED life at room temperature, and a hot panel will lose brightness faster than that number suggests.

## Sources

- ESPHome LEDC output (1 kHz default, recommended frequency / bit depth table, shared timers): https://esphome.io/components/output/ledc/
- ESPHome Waveshare CH32V003 I/O expander (PWM output, `safe_pwm_levels`, `power_supply` example): https://esphome.io/components/waveshare_io_ch32v003/
- Riverdi, Driving Circuit Design Guide for backlight and DC-DC, 7" HB IPS RGB series, Rev 1.2: https://download.riverdi.com/RVT70HSTNWN00/AN_RVT70HSTNWN00_Rev1.2.pdf
- Newhaven NHD-7.0-800480AF-ASXP datasheet: https://newhavendisplay.com/content/specs/NHD-7.0-800480AF-ASXP.pdf

I work at a display maker, so take that into account. We build high-brightness TFT modules among other things, but nothing here is specific to our parts. It should apply to any panel with a boost LED driver on the backlight.
