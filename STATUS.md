# Status and change log

Part of the [MSI Optix G27C2 protocol reference](README.md). One row per code or area; the tags are explained in the
README.

## DDC/CI and EDID

| Code / area | Name | Read | Write | Open |
|---|---|---|---|---|
| physical monitor | handle from `GetPhysicalMonitorsFromHMONITOR` | value 0, valid (OBSERVED) | | |
| session | where DDC/CI works | user session only; none in session 0 (OBSERVED) | | |
| capability string | `GetCapabilitiesStringLength` | fails every time with an I2C error (OBSERVED) | | why |
| reliability | I2C transmit errors | about one read in four fails; retry works (OBSERVED) | sets fail without idle time; 50 ms after the end of a command works (OBSERVED) | behaviour while the monitor sleeps |
| `0x0B` / `0x0C` | colour temperature increment / request | 50 K; 6500 K, 9300 K (OBSERVED) | not tried | writability of `0x0C` |
| `0x10` | brightness | yes | **yes** (OBSERVED 2026-11-01) | persistence across a power-button cycle |
| `0x12` | contrast | yes | **yes** (OBSERVED 2026-11-02) | as above |
| `0x14` | colour preset: Cool `04`, Warm `05`, User `06` | yes; a late update after an OSD change once | **ignored** (OBSERVED 2026-11-02) | other values |
| `0x16` / `0x18` / `0x1A` | User RGB gains, 0-100 | yes (R mapped to the OSD) | not tried | writability |
| `0x60` | input | `3` on DisplayPort | not tried | HDMI numbers |
| `0x62` | volume | `50` | not tried | |
| `0x6C` / `0x6E` / `0x70` | black levels | `50` each | not tried | OSD mapping |
| `0xD6` | power mode | `1` (on) | not tried; hazardous | |
| `0xDF` | MCCS version | `0x0201` = 2.1 | | |
| `0xE0`-`0xFF` | manufacturer range | `0xE1`, `0xFF` answer; most "not supported"; 11 codes inconclusive | | meanings; re-read the inconclusive ones |
| EDID | identity, timings, audio | complete (HOST) | | |

## Change log

- **2026-11-04, first edition.** Written from reads and writes through Windows' DDC/CI API by independently written
  code (2026-11-01 to 2026-11-04), an OSD-correlation session for the colour presets (2026-11-02), Windows' stored
  EDID, and the public sources credited in the README.
