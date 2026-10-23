# CUSTOM init commands longer than 127 bytes — "Malformed init sequence"

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). This is **not bench-tested here**. I'm writing it down because vendor init dumps for MIPI / QSPI panels often carry one register write with well over 127 data bytes, and ESPHome's current readers quietly truncate that length.

## What it looks like

You paste a long vendor `init_sequence` under `model: CUSTOM` (or an external component that flattens the same format). Compile is fine. At boot the log says **Malformed init sequence** and the display component marks failed — or it does not fail, but later commands are shifted because the length byte was clipped and the reader ate the next command as payload.

That pattern shows up on panels whose factory tables put a big gamma / power / page dump in a single DCS write. Short stock models (ST7789V, ILI9341, …) rarely hit it. Long CUSTOM dumps do.

## Why

The flattened init format stores each step as `cmd`, then a length byte `x`, then `x` argument bytes (or a delay flag). On `dev` today several drivers still decode the length like this (snippet from `esphome/components/mipi_dsi/mipi_dsi.cpp` on `dev`, same shape in `mipi_rgb`, `epaper_spi`, `mipi_spi`, `qspi_dbi`):

```cpp
uint8_t cmd = seq[index++];
uint8_t x = seq[index++];
if (x == DELAY_FLAG) {
  // delay path
} else {
  uint8_t num_args = x & 0x7F;
  if (len - index < num_args) {
    this->mark_failed(LOG_STR("Malformed init sequence"));
    return;
  }
  // ... tx_param(cmd, ptr, num_args); index += num_args;
}
```

`x & 0x7F` drops the high bit. So a command that really has 128–253 data bytes is read as `length - 128`. The YAML / `mipi.map_sequence` side already accepts up to 253 bytes; the C++ readers never did. Mismatch → Malformed, or a truncated write and a corrupted stream for everything after.

## Upstream PRs (closed, not merged)

[esphome/esphome#21440](https://github.com/esphome/esphome/pull/21440) adds a shared `InitSequenceReader` that honours the full length. [esphome/esphome#21441](https://github.com/esphome/esphome/pull/21441) moves `st7305`, `mipi_spi`, `mipi_rgb`, `mipi_dsi`, and `epaper_spi` onto that reader (and notes the >127 bug). Both closed unmerged around **2026-10-23 04:58 UTC**; #21441 was based on the feature branch from #21440, not on `dev` alone. I re-checked `dev` `mipi_dsi.cpp` after that close — the `& 0x7F` mask is still there. I am not claiming when (or if) a rebased version lands.

## Practical workaround until then

If your vendor dump has one command with more than 127 data bytes, **split that dump into several shorter commands** when the datasheet allows (same register address / page select, smaller chunks). Keep each length byte ≤ 127 so the current readers stay honest.

If you cannot split: watch for `Malformed init sequence` right after setup, and for "works until command N then garbage" when N sits on a long write. Do not assume the YAML validator caught it — it can accept a length the firmware still masks.

I am not publishing a patch fork here. Point people at #21440 / #21441 if they want the shared reader; for bring-up today, shorter commands are the reliable path.

## Disclosure

I work for Display Engineer. This note restates public code and PR text from ESPHome `dev` and #21440/#21441. We did not run a >127-byte CUSTOM sequence on our bench for this commit. At most one link above points at our site.
